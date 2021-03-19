# Building a Recommender with Apache Mahout on Amazon Elastic MapReduce (EMR) - Tutorial

This is a tutorial on working with mahout based on the original tutorial posted aws blogs [here](https://aws.amazon.com/blogs/big-data/building-a-recommender-with-apache-mahout-on-amazon-elastic-mapreduce-emr/). The steps and given scripts are modified in order to work with newer versions of EMR and python. 

Please follow the original tutorial to grab the theory and more insights on this. In this tutorial will run only through the steps.

This will be carried out in a Linux environment and all the commands associated are with linux terminal.

## Pre-requisits

1. Amazon AWS Account and basic knowledge on aws services
2. Basic knowledge on linux terminal
3. Basic knowledge on Elastic-Map-reduce and HDFS File System

## 1. Installing and Configuring the awscli

Run following commands to install awscli
```bash
$ sudo apt-get update
$ sudo apt-get install awscli
```
To configure the awscli to your instance run the following command and give the requested credentials.

OR either you can navigate to aws configuration file location(by default ~/.aws location) and edit it.
```bash
$ aws configure

AWS Access Key ID : *************
AWS Secret Access Key : **************
Default region name : us-east-1 
Default output format : json
```
To check the connection use the command `aws iam list-users`. If it replies with a json as follows you are connected.

```bash
{
    "Users": [
        {
            "Path": "/",
            "UserName": "*****",
            "UserId": "******************",
            "Arn": "******************",
            "CreateDate": "************"
        }
    ]
}

```

## 2. Creating a key pair for connecting to an emr
To connect to an emr you need to have a key pair. That can be created and save into the Downloads folder by running following command. Then make the key read only by running `chmod 400` 

```bash
$ aws ec2 create-key-pair --key-name your-key-name --query 'KeyMaterial' --output text > ~/Downloads/your-key-name.pem
$ chmod 400 ~/Downloads/your-key-name.pem
```
## 3. Creating an s3 bucket for saving emr cluster logs
This is not an essential step for this tutorial. This s3 bucket would be used only to save the logs. You can have it in the default logs folder instead.
```bash
$ aws s3api create-bucket --bucket your-bucket-name --region us-east-1
```
## 4. Creating the EMR
We'll be creating a EMR cluster with the following profiles. You can select configuration of your own considering the charges for the clusters. But as the Mahout can take lot of resources make sure the configurations you select are enough for this exercise. Remember to replace the key name and s3 bucket with what you created earlier. 
```bash
$ aws emr create-cluster \
--ec2-attributes '{"KeyName":"your-key-name","InstanceProfile":"EMR_EC2_DefaultRole"}' \
--service-role EMR_DefaultRole \
--enable-debugging \
--release-label emr-5.32.0 \
--log-uri 's3n://your-bucket-name/logs/' \
--name 'mahout-tutorial' \
--instance-groups '[{"InstanceCount":2,"InstanceGroupType":"CORE","InstanceType":"m5.xlarge","Name":"Core Instance Group"},{"InstanceCount":1,"InstanceGroupType":"MASTER","InstanceType":"m5.xlarge","Name":"Master Instance Group"}]' \
--scale-down-behavior TERMINATE_AT_TASK_COMPLETION \
--region us-east-1
```
If the cluster is Starting you should get a response with a json text having `ClusterId` and `ClusterArn`. You can use the command `$ aws emr list-clusters --active` to see the cluster state. Once it is in *Waiting* state rest of the exercise can be started.