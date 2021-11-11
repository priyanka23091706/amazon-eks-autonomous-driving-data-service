# Data Service for ADAS and ADS Development

## Overview

This is an example of a data service typically used in advanced driver assistance systems (ADAS), and automated driving systems (ADS) development. The data service is composed from following modular AWS services: [Amazon Elastic Kubernetes Service (EKS)](https://aws.amazon.com/eks/), [Amazon Managed Streaming for Apache Kafka (MSK)](https://aws.amazon.com/msk/), [Amazon Redshift](https://aws.amazon.com/redshift), [Amazon FSx](https://aws.amazon.com/fsx/), [Amazon Elastic File System (EFS)](https://aws.amazon.com/efs/), [AWS Batch](https://aws.amazon.com/batch/), [AWS Step Functions](https://aws.amazon.com/step-functions/), [AWS Secrets Manager](https://aws.amazon.com/secrets-manager/), [Amazon Glue](https://aws.amazon.com/glue/), [Amazon Fargate](https://aws.amazon.com/fargate/), [Amazon Elastic Container Registry (ECR)](https://aws.amazon.com/ecr/), [Amazon Virtual Private Cloud (VPC)](https://aws.amazon.com/vpc/), [Amazon EC2](https://aws.amazon.com/ec2/), and [Amazon S3](https://aws.amazon.com/s3/).

The typical use case addressed by this data service is to serve sensor data from a specified drive scene of interest, either in *batch mode* as a single [```rosbag```](http://wiki.ros.org/rosbag)  file, or in *streaming mode* as an ordered series of ```rosbag``` files, whereby each ```rosbag``` file contains the drive scene data for a single time step, which by default is 1 second. Each ```rosbag``` file is dynamically composed from the drive scene data stored in [Amazon S3](https://aws.amazon.com/free/storage/s3), using the meta-data stored in [Amazon Redshift](https://aws.amazon.com/redshift/). 


## Key concepts

The data service runs in [Kubernetes Pods](https://kubernetes.io/docs/concepts/workloads/pods/) in an [Amazon EKS](https://aws.amazon.com/eks/) cluster configured to use [Horizontal Pod Autoscaler](https://docs.aws.amazon.com/eks/latest/userguide/horizontal-pod-autoscaler.html) and EKS [Cluster Autoscaler](https://docs.aws.amazon.com/eks/latest/userguide/cluster-autoscaler.html). 

An [Amazon Managed Service For Apache Kafka](https://aws.amazon.com/msk/) (MSK) cluster provides the communication channel between the data service, and the client. The data service implements a *request-response* paradigm over Apache Kafka topics. However, the response data is not sent back over the Kafka topics. Instead, the data service stages the response data in Amazon S3, Amazon FSx, or Amazon EFS, as specified in the data client request, and the location of the response data is sent back to the data client over the Kafka topics. The data client directly reads the response data from its staged location.

### Data client request
Concretely, imagine the data client wants to request drive scene  data in ```rosbag``` file format from [A2D2 autonomous driving dataset](https://www.a2d2.audi/a2d2/en.html) for vehicle id ```a2d2```, drive scene id ```20190401145936```, starting at timestamp ```1554121593909500``` (microseconds) , and stopping at timestamp ```1554122334971448``` (microseconds). The data client wants the response to include data **only** from the ```front-left camera``` in ```sensor_msgs/Image``` [ROS](https://www.ros.org/) data type, and the ```front-left lidar``` in ```sensor_msgs/PointCloud2``` ROS data type. The data client wants the response data to be *streamed* back chunked in series of ```rosbag``` files, each file spanning ```1000000``` microseconds of the drive scene. Finally, the data client wants the response ```rosbag``` files to be stored on a shared Amazon FSx file system. 

The data client can encode such a data request using the JSON object shown below, and send it to the (imaginary) Kafka cluster endpoint ```b-1.msk-cluster-1:9092,b-2.msk-cluster-1:9092``` on the Apache Kafka topic named ```a2d2```:

```
{
	"servers": "b-1.msk-cluster-1:9092,b-2.msk-cluster-1:9092",
	"requests": [{
		"kafka_topic": "a2d2", 
		"vehicle_id": "a2d2",
		"scene_id": "20190401145936",
		"sensor_id": ["lidar/front_left", "camera/front_left"],
		"start_ts": 1554121593909500, 
		"stop_ts": 1554122334971448,
		"ros_topic": {"lidar/front_left": "/a2d2/lidar/front_left", 
				"camera/front_left": "/a2d2/camera/front_left"},
		"data_type": {"lidar/front_left": "sensor_msgs/PointCloud2",
				"camera/front_left": "sensor_msgs/Image"},
		"step": 1000000,
		"accept": "fsx/multipart/rosbag",
		"preview": false
	}]
}
```

For a detailed description of each request field shown in the example above, see [data request fields](#RequestFields) below.

## Tutorial step by step guide

### Overview
In this tutorial, we use [A2D2 autonomous driving dataset](https://www.a2d2.audi/a2d2/en.html). The high-level outline of this tutorial is as follows:

* Prerequisites
* Configure the data service
* Prepare the A2D2 data
* Run the data service
* Run the data service client


### Prerequisites
This tutorial assumes you have an [AWS Account](https://aws.amazon.com/account/), and you have [Administrator job function](https://docs.aws.amazon.com/IAM/latest/UserGuide/access_policies_job-functions.html) access to the AWS Management Console.

To get started:

* Select your [AWS Region](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/using-regions-availability-zones.html). The AWS Regions supported by this project include, us-east-1, us-east-2, us-west-2, eu-west-1, eu-central-1, ap-southeast-1, ap-southeast-2, ap-northeast-1, ap-northeast-2, and ap-south-1. Note that not all Amazon EC2 instance types are available in all [AWS Availability Zones](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/using-regions-availability-zones.html) in an AWS Region.
* Subscribe to [Ubuntu Pro 18.04 LTS](https://aws.amazon.com/marketplace/pp/Amazon-Web-Services-Ubuntu-Pro-1804-LTS/B0821T9RL2) and [Ubuntu Pro 20.04 LTS](https://aws.amazon.com/marketplace/pp/Amazon-Web-Services-Ubuntu-Pro-2004-LTS/B087L1R4G4).
* If you do not already have an Amazon EC2 key pair, [create a new Amazon EC2 key pair](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-key-pairs.html#prepare-key-pair). You will need the key pair name to specify the ```KeyName``` parameter when creating the AWS CloudFormation stack below. 
* You will need an [Amazon S3](https://aws.amazon.com/s3/) bucket. If you don't have one, [create a new Amazon S3 bucket](https://docs.aws.amazon.com/AmazonS3/latest/userguide/create-bucket-overview.html) in the AWS region of your choice. You will use the S3 bucket name to specify the ```S3Bucket``` parameter in the stack. 
* Run ```curl ifconfig.co``` on your laptop and note your public IP address. This will be the IP address you will need to specify ```DesktopAccessCIDR``` parameter in the stack.

### Configure the data service

#### Create AWS CloudFormation Stack
The [AWS CloudFormation](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/Welcome.html) template ```cfn/mozart.yml``` in this repository creates [AWS Identity and Access Management (IAM)](https://aws.amazon.com/iam/) resources, so when you [create the CloudFormation Stack using the console](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/cfn-console-create-stack.html), you must check 
**I acknowledge that AWS CloudFormation might create IAM resources.** 

Create a new AWS CloudFormation stack using the ```cfn/mozart.yml``` template. The stack input parameters you must specify are described below:

| Parameter Name | Parameter Description |
| --- | ----------- |
| KeyPairName | This is a *required* parameter whereby you select the Amazon EC2 key pair name used for SSH access to the desktop. You must have access to the selected key pair's private key to connect to your desktop. |
| RedshiftMasterUserPassword | This is a *required* parameter whereby you specify the Redshift database master user password.|
| RemoteAccessCIDR | This is a *required* parameter whereby you specify the public IP CIDR range from where you need remote access to your graphics desktop, e.g. 1.2.3.4/32, or 7.8.0.0/16. |
| S3Bucket | This is a *required* parameter whereby you specify the name of the Amazon S3 bucket to store your data. **The S3 bucket must already exist.** |

For all other stack input parameters, default values are recommended during first walkthrough. See complete list of all the [template input parameters](#InputParams) below. 


#### Connect to the graphics desktop using SSH

* Once the stack status in CloudFormation console is ```CREATE_COMPLETE```, find the desktop instance launched in your stack in the Amazon EC2 console, and [connect to the instance using SSH](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/AccessingInstancesLinux.html) as user ```ubuntu```, using your SSH key pair.
* When you connect to the desktop using SSH, and you see the message ```"Cloud init in progress. Machine will REBOOT after cloud init is complete!!"```, disconnect and try later after about 20 minutes. The desktop installs the NICE DCV server on first-time startup, and reboots after the install is complete.
* If you see the message ```NICE DCV server is enabled!```, run the command ```sudo passwd ubuntu``` to set a new password for user ```ubuntu```. Now you are ready to connect to the desktop using the [NICE DCV client](https://docs.aws.amazon.com/dcv/latest/userguide/client.html)

#### Connect to the graphics desktop using NICE DCV Client
* Download and install the [NICE DCV client](https://docs.aws.amazon.com/dcv/latest/userguide/client.html) on your laptop.
* Use the NICE DCV Client to login to the desktop as user ```ubuntu```
* When you first login to the desktop using the NICE DCV client, you will be asked if you would like to upgrade the OS version. **Do not upgrade the OS version** .

Now you are ready to proceed with the following steps. For all the commands in this tutorial, we assume the *working directory* to be ``` ~/amazon-eks-autonomous-driving-data-service``` on the graphics desktop. 

#### Configure EKS cluster access

In this step, you need the [AWS credentials](https://docs.aws.amazon.com/general/latest/gr/aws-sec-cred-types.html#access-keys-and-secret-access-keys) for the IAM user or role you used to create the AWS CloudFormation stack above. The AWS credentials are used *one-time* to enable EKS cluster access from the graphics desktop, and are removed at the end of this step. In the *working directory*, run the command:

		./scripts/configure-eks-auth.sh

At the end of this command output, you should see ```AWS Credentials Removed```. 

#### Setup EKS cluster environment

To setup the eks cluster environment, in the *working directory*, run the command:

		./scripts/setup-dev.sh

This step also builds and pushes the data service container image into [Amazon ECR](https://aws.amazon.com/ecr/).

### Prepare the A2D2 data

Before we can run the A2D2 data service, we need to extract the raw A2D2 data into your S3 bucket, extract the metadata from the raw data, and upload the metadata into the Redshift cluster. We execute these three steps using an [AWS Step Functions](https://docs.aws.amazon.com/step-functions/latest/dg/welcome.html) state machine. To create and execute the AWS Step Functions state machine, execute the following command in the *working directory*:

		./scripts/a2d2-etl-steps.sh

Note the ```executionArn``` of the state machine execution in the output of the previous command. To check the status the status of the execution, use following command, replacing ```executionArn``` below with your value:

	aws stepfunctions describe-execution --execution-arn executionArn

The state machine execution time varies, and may take approximately 8 - 12 hours. 

### Run the data service

*For best performance, [preload A2D2 data from S3 to FSx](#PreloadFSx). For a quick preview of the data service, you may proceed with the step below.*

The data service is deployed using an [Helm Chart](https://helm.sh/docs/topics/charts/), and runs as a ```kubernetes deployment``` in EKS. To start the data service, execute the following command in the *working directory*:

		helm install --debug a2d2-data-service ./a2d2/charts/a2d2-data-service/

To verify that the ```a2d2-data-service``` deployment is running, execute the command:

		kubectl get pods -n a2d2
		
You can **stop** the data service by executing:

		helm delete a2d2-data-service

#### Raw data input data source, and response data staging

The data service can be configured to input raw data from S3, FSx (see [Preload A2D2 data from S3 to FSx](#PreloadFSx) ), or EFS (see [Preload A2D2 data from S3 to EFS](#PreloadEFS) ).  Similarly, the data client can specify the ```accept``` field in the ```request``` to request that the response data be staged on S3, FSx, or EFS. The raw data input source is fixed when the data service is deployed. However, the response data staging option is specified in the data client request, and is therefore dynamic. 

Below is the Helm chart configuration in [```values.yaml```](a2d2/charts/a2d2-data-service/values.yaml) for various raw input data source options, with recommended Kubernetes resource requests for pod ```memory``` and ```cpu```:

 Raw input data source | ```values.yaml``` Configuration |
| --- | ----------- |
| ```fsx``` (default) | ```a2d2.requests.memory: "72Gi"``` <br> ```a2d2.requests.cpu: "8000m"``` <br> ```configMap.data_store.input: "fsx"```|
| ```efs```  | ```a2d2.requests.memory: "32Gi"``` <br> ```a2d2.requests.cpu: "1000m"``` <br> ```configMap.data_store.input: "efs"```|
| ```s3```  | ```a2d2.requests.memory: "8Gi"``` <br> ```a2d2.requests.cpu: "1000m"``` <br> ```configMap.data_store.input: "s3"```|

For matching response data staging options in data client request, see ```requests.accept``` field in [data request fields](#RequestFields). The recommended response data staging option for ```fsx``` raw data source is```"accept": "fsx/multipart/rosbag"```, for ```efs``` raw data source is```"accept": "efs/multipart/rosbag"```, and for ```s3``` raw data source is```"accept": "s3/multipart/rosbag"```

### Run the data service client


To visualize the response data requested by the A2D2 data client, we will use [rviz](http://wiki.ros.org/rviz) tool on the graphics desktop. Open a terminal on the desktop, and run ```rviz```. In the ```rviz``` tool, set ```/home/ubuntu/amazon-eks-autonomous-driving-data-service/a2d2/config/a2d2.rviz``` as the ```rviz``` config file. You should see ```rviz``` tool now configured with two areas, one for visualizing image data, and the other for visualizing point cloud data.
  
To run the data client, execute the following command in the *working directory*:

		python ./a2d2/src/data_client.py \
			--config ./a2d2/config/c-config-ex1.json 1>/tmp/a.out 2>&1 & 

After a brief delay, you should be able to preview the response data in the ```rviz``` tool To preview data from a different drive scene, execute:

		python ./a2d2/src/data_client.py \
			--config ./a2d2/config/c-config-ex2.json 1>/tmp/a.out 2>&1 & 


You can set ```"preview": false``` in the data client config file, and run the above command to view the complete response, but before you do that, we recommend that for best performance, [preload A2D2 data from S3 to FSx](#PreloadFSx).
			
### Killing data service client

Killing the data client does not stop the data service from producing and sending the ```rosbag``` files to the client, as the data information is being sent asynchronously. If the data client is killed by the user, the data service will continue to stage the response data, and the data will remain stored on the response data staging option specified in the data client request.

If you kill the data service client before it exits normally, be sure to clean up all the processes spawned by the data client. 


## <a name="Reference"></a> Reference

### <a name="RequestFields"></a> Data client request fields
Below, we explain the semantics of the various fields in the data client request JSON object.

| Request field name | Request field description |
| --- | ----------- |
| ```servers``` | The ```servers``` identify the [AWS MSK](https://aws.amazon.com/msk/) Kafka cluster endpoint. |
| ```requests```| The JSON document sent by the client to the data service must include an array of one or more data ```requests``` for drive scene data. |
| ```requests.kafka_topic``` | The ```kafka_topic``` specifies the Kafka topic on which the data request is sent from the client to the data service. The data service is listening on the topic. |
| ```requests.vehicle_id``` | The ```vehicle_id``` is used to identify the relevant drive scene dataset. |
| ```requests.scene_id```  | The ```scene_id``` identifies the drive scene of interest, which in this example is ```20190401145936```, which in this example is a string representing the date and time of the drive scene, but in general could be any unique value. |
| ```requests.start_ts``` | The ```start_ts``` (microseconds) specifies the start timestamp for the drive scene data request. |
| ```requests.stop_ts``` | The ```stop_ts``` (microseconds) specifies the stop timestamp for the drive scene data request. |
| ```requests.ros_topic``` | The ```ros_topic``` is a dictionary from ```sensor ids``` in the vehicle to ```ros``` topics.|
| ```requests.data_type```| The ```data_type``` is a dictionary from ```sensor ids``` to ```ros``` data types.  |
| ```requests.step``` | The ```step``` is the discreet time interval (microseconds) used to discretize the timespan between ```start_ts``` and ```stop_ts```. If ```requests.accept``` value contains ```multipart```, the data service responds with a ```rosbag``` file for each discreet ```step```: See [possible values](#AcceptValues) below. |
| ```requests.accept``` | The ```accept``` specifies the response data staging format acceptable to the client: See [possible values](#AcceptValues) below. |
| ```requests.image``` | (Optional) The value ```undistorted``` undistorts the camera image. Undistoring an image slows down the image frame rate. |
| ```requests.lidar_view``` | (Opitonal) The value ```vehicle``` transforms lidar points to ```vehicle``` frame of reference view. |
|```requests.preview```| If the ```preview``` field is set to ```true```, the data service returns requested data over a single time ```step``` starting from ```start_ts``` , and ignores the ```stop_ts```.|

#### <a name="AcceptValues"></a>  Possible ```requests.accept``` field values

Below we describe the possible values for ```requests.accept``` field:

| ```requests.accept``` value | Description |
| --- | ----------- |
| ```fsx/multipart/rosbag``` | Stage response data on shared Amazon FSx file-system as  discretized ```rosbag``` chunks |
| ```efs/multipart/rosbag``` | Stage response data on shared Amazon EFS file-system as  discretized ```rosbag``` chunks|
|```s3/multipart/rosbag```| Stage response data on Amazon S3 as  discretized ```rosbag``` chunks|
| ```fsx/singlepart/rosbag``` | Stage response data on shared Amazon FSx file-system as  single ```rosbag``` |
| ```efs/singlepart/rosbag``` | Stage response data on on shared Amazon EFS file-system as  single ```rosbag```|
|```s3/singlepart/rosbag```| Stage response data on Amazon S3 as  single ```rosbag```|
| ```manifest``` | Return a manifest of meta-data containing S3 paths to the raw data: ```manifest``` is returned directly over the Kafka response topic, and is not staged |


### <a name="InputParams"></a> AWS CloudFormation template input parameters
This repository provides an [AWS CloudFormation](https://aws.amazon.com/cloudformation/) template that is used to create the required stack.

Below, we describe the AWS CloudFormation [template](cfn/mozart.yml) input parameters. Desktop below refers to the NICE DCV enabled high-performance graphics desktop that acts as the data service client in this tutorial.

| Parameter Name | Parameter Description |
| --- | ----------- |
| DesktopInstanceType | This is a **required** parameter whereby you select an Amazon EC2 instance type for the desktop running in AWS cloud. Default value is ```g3s.xlarge```. |
| DesktopEbsVolumeSize | This is a **required** parameter whereby you specify the size of the root EBS volume (default size is 200 GB) on the desktop. Typically, the default size is sufficient.|
| DesktopEbsVolumeType | This is a **required** parameter whereby you select the [EBS volume type](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ebs-volume-types.html) (default is gp3). |
| DesktopHasPublicIpAddress | This is a **required** parameter whereby you select whether a Public Ip Address be associated with the Desktop.  Default value is ```true```.|
| EKSEncryptSecrets | This is a **required** parameter whereby you select if encryption of EKS secrets is ```Enabled```. Default value is ```Enabled```.|
| EKSEncryptSecretsKmsKeyArn | This is an *optional* advanced parameter whereby you specify the [AWS KMS](https://aws.amazon.com/kms/) key ARN that is used to encrypt EKS secrets. Leave blank to create a new KMS key.|
| EKSNodeGroupInstanceType | This is a **required** parameter whereby you select EKS Node group EC2 instance type. Default value is ```r5n.8xlarge```.|
| EKSNodeVolumeSizeGiB | This is a **required** parameter whereby you specify EKS Node group instance EBS volume size. Default value is 200 GiB.|
| EKSNodeGroupMinSize | This is a **required** parameter whereby you specify EKS Node group minimum size. Default value is 1 node.|
| EKSNodeGroupMaxSize | This is a **required** parameter whereby you specify EKS Node group maximum size. Default value is 8 nodes.|
| EKSNodeGroupDesiredSize | This is a **required** parameter whereby you specify EKS Node group initial desired size. Default value is 2 nodes.|
| FSxStorageCapacityGiB |  This is a **required** parameter whereby you specify the FSx Storage capacity, which must be in multiples of 3600 GiB. Default value is 7200 GiB.|
| FSxS3ImportPrefix | This is an *optional* advanced parameter whereby you specify FSx S3 bucket path prefix for importing data from S3 bucket. Leave blank to import the complete bucket.|
| KeyPairName | This is a **required** parameter whereby you select the Amazon EC2 key pair name used for SSH access to the desktop. You must have access to the selected key pair's private key to connect to your desktop. |
| KubectlVersion | This is a **required** parameter whereby you specify EKS ```kubectl``` version. Default value is ```1.21.2/2021-07-05```. |
| KubernetesVersion | This is a **required** parameter whereby you specify EKS cluster version. Default value is ```1.21```. |
| MSKBrokerNodeType | This is a **required** parameter whereby you specify the type of node to be provisioned for AWS MSK Broker. |
| MSKNumberOfNodes | This is a **required** parameter whereby you specify the number of MSK Broker nodes, which must be >= 2. |
| PrivateSubnet1CIDR | This is a **required** parameter whereby you specify the Private Subnet1 CIDR in Vpc CIDR. Default value is ```172.30.64.0/18```.|
| PrivateSubnet2CIDR | This is a **required** parameter whereby you specify the Private Subnet2 CIDR in Vpc CIDR. Default value is ```172.30.128.0/18```.|
| PrivateSubnet3CIDR | This is a **required** parameter whereby you specify the Private Subnet3 CIDR in Vpc CIDR. Default value is ```172.30.192.0/18```.|
| PublicSubnet1CIDR | This is a **required** parameter whereby you specify the Public Subnet1 CIDR  in Vpc CIDR. Default value is ```172.30.0.0/24```.|
| PublicSubnet2CIDR | This is a **required** parameter whereby you specify the Public Subnet2 CIDR  in Vpc CIDR. Default value is ```172.30.1.0/24```.|
| PublicSubnet3CIDR | This is a **required** parameter whereby you specify the Public Subnet3 CIDR  in Vpc CIDR. Default value is ```172.30.2.0/24```.|
| RedshiftDatabaseName | This is a **required** parameter whereby you specify the name of the Redshift database. Default value is ```mozart```.|
| RedshiftMasterUsername | This is a **required** parameter whereby you specify the name Redshift Master user name. Default value is ```admin```.|
| RedshiftMasterUserPassword | This is a **required** parameter whereby you specify the name Redshift Master user password.|
| RedshiftNodeType | This is a **required** parameter whereby you specify the type of node to be provisioned for Redshift cluster. Default value is ```ra3.4xlarge```.|
| RedshiftNumberOfNodes | This is a **required** parameter whereby you specify the number of compute nodes in the Redshift cluster, which must be >= 2.|
| RedshiftPortNumber | This is a **required** parameter whereby you specify the port number on which the Redshift cluster accepts incoming connections. Default value is ```5439```.|
| RemoteAccessCIDR | This parameter specifies the public IP CIDR range from where you need remote access to your client desktop, e.g. 1.2.3.4/32, or 7.8.0.0/16. |
| S3Bucket | This is a **required** parameter whereby you specify the name of the Amazon S3 bucket to store your data. |
| UbuntuAMI | This is an *optional* advanced parameter whereby you specify Ubuntu AMI (18.04 or 20.04).|
| VpcCIDR | This is a **required** parameter whereby you specify the [Amazon VPC](https://aws.amazon.com/vpc/?vpc-blogs.sort-by=item.additionalFields.createdDate&vpc-blogs.sort-order=desc) CIDR for the VPC created in the stack. Default value is 172.30.0.0/16. If you change this value, all the subnet parameters above may need to be set, as well.|

### <a name="PreloadFSx"></a>  Preload A2D2 data from S3 to FSx  

*This step can be executed anytime after "Configure the data service" step has been executed*

Amazon FSx for Lustre automatically lazy loads data from the configured S3 bucket. Therefore, this step is strictly a performance optimization step . However, for maximal performance, it is *highly recommended.* Execute following command to start preloading data from your S3 bucket to the FSx file-system:

	kubectl apply -n a2d2 -f a2d2/fsx/stage-data-a2d2.yaml
	
To check if the step is complete, execute:

	kubectl get pods stage-fsx-a2d2 -n a2d2

If the pod is still ```Running```, the step has not yet completed. This step takes approximately 5 hours to complete.


### <a name="PreloadEFS"></a> Preload A2D2 data from S3 to EFS 

*This step can be executed anytime after "Configure the data service" step has been executed*

This step is required **only** if you plan to configure the data service to use EFS as the raw data input source, otherwise, it may be safely skipped. Execute following command to start preloading data from your S3 bucket to the EFS file-system:

	kubectl apply -n a2d2 -f a2d2/efs/stage-data-a2d2.yaml

To check if the step is complete, execute:

	kubectl get pods stage-efs-a2d2 -n a2d2

If the pod is still ```Running```, the step has not yet completed. This step takes approximately 6.5 hours to complete.





request