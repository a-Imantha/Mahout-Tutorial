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
--applications Name=Hadoop Name=Hive Name=Hue Name=Mahout Name=Pig Name=Tez \
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

With the default security group it will not allow ssh connection with the existing inbound rules. So the inbound rule should be added following the below steps.
1. First find the ip address of your machine
```bash
$ curl https://checkip.amazonaws.com
```
2. Use the following command to find the information on cluster. Fill the cluster id parameter from your cluster id. Look for the parameter *EmrManagedMasterSecurityGroup* and use this as the security group id.
```bash
$ aws emr describe-cluster --cluster-id j-1W********
```
3. Fill the parameters from above steps and Add the new inbound rule. ssh use tcp port 22 and keep them unchanged.
```bash
$ aws ec2 authorize-security-group-ingress \
--group-id sg-03********* \
--protocol tcp \
--port 22 \
--cidr 17*.***.**.***/32
```
## 5. Connecting to the Cluster Master Node with SSH.
You can run the following command to connect to the master node with ssh
```bash
$ aws emr ssh --cluster-id j-1W******* --key-pair-file ~/Downloads/your-key-name.pem
```
Once you are succesfully logged in you should see the emr instance loading on your terminal
```bash

       __|  __|_  )
       _|  (     /   Amazon Linux 2 AMI
      ___|\___|___|

https://aws.amazon.com/amazon-linux-2/
38 package(s) needed for security, out of 76 available
Run "sudo yum update" to apply all updates.
                                                                    
EEEEEEEEEEEEEEEEEEEE MMMMMMMM           MMMMMMMM RRRRRRRRRRRRRRR    
E::::::::::::::::::E M:::::::M         M:::::::M R::::::::::::::R   
EE:::::EEEEEEEEE:::E M::::::::M       M::::::::M R:::::RRRRRR:::::R 
  E::::E       EEEEE M:::::::::M     M:::::::::M RR::::R      R::::R
  E::::E             M::::::M:::M   M:::M::::::M   R:::R      R::::R
  E:::::EEEEEEEEEE   M:::::M M:::M M:::M M:::::M   R:::RRRRRR:::::R 
  E::::::::::::::E   M:::::M  M:::M:::M  M:::::M   R:::::::::::RR   
  E:::::EEEEEEEEEE   M:::::M   M:::::M   M:::::M   R:::RRRRRR::::R  
  E::::E             M:::::M    M:::M    M:::::M   R:::R      R::::R
  E::::E       EEEEE M:::::M     MMM     M:::::M   R:::R      R::::R
EE:::::EEEEEEEE::::E M:::::M             M:::::M   R:::R      R::::R
E::::::::::::::::::E M:::::M             M:::::M RR::::R      R::::R
EEEEEEEEEEEEEEEEEEEE MMMMMMM             MMMMMMM RRRRRRR      RRRRRR
```
## 6. Downloading and exploring the MovieLense Data

This will be the dataset analysed in here. we should download the required files from http://files.grouplens.org/datasets/movielens/ml-1m.zip We will create a folder and then download the files into it.
```bash
$ mkdir mahout-tut
$ cd mahout-tut
$ wget http://files.grouplens.org/datasets/movielens/ml-1m.zip
$ unzip ml-1m.zip
```
After downloading and unziping data you can use the cat or nano commands to view the files
```bash
nano ml-1m/ratings.dat
```
Sample of the output:
```
1::1193::5::978300760
1::661::3::978302109
1::914::3::978301968
1::3408::4::978300275
1::2355::5::978824291
1::1197::3::978302268
1::1287::5::978302039
```

## 7. Creating a CSV out of the ratings.dat
In order to create a csv will need to replace '::' with ','. With the following command we will be replacing double colans with comma and save the file as *ratings.csv* in the current directory.
```bash
$ cat ml-1m/ratings.dat | sed 's/::/,/g' | cut -f1-3 -d, > ratings.csv
$ nano ratings.csv

```
## 8. Put the Rating file to HDFS
Our expectation is to train a machine algorithm with the created csv. Mahout will train the model through map reduce. To do this we will be putting the csv to HDFS with following command.
```bash
$ hadoop fs -put ratings.csv ratings.csv
```
You can verify the file is in hadoop by running the list command
```bash
$ hadoop fs -ls
```
## 9. Run the Recommender Job
The recommender will be trained with the ratings.csv to recommend movies to the respective user. For this recommenditembased on Mahout will be used.
```bash
$ mahout recommenditembased --input /ratings.csv \
--output recommendations \
--numRecommendations 10 \
--outputPathForSimilarityMatrix similarity-matrix \
--similarityClassname SIMILARITY_COSINE
```
This will run several map-reduce steps to train the recommender. After that we can extract the created output file from hdfs. With the following command we can see the file and this gives 10 movie reccomendations to the user with how strong the reccomendation is to a given user. A good explanation on the output can be found in the original [linked](https://aws.amazon.com/blogs/big-data/building-a-recommender-with-apache-mahout-on-amazon-elastic-mapreduce-emr/)document.

```bash
$ hadoop fs -ls recommendations
$ hadoop fs -cat recommendations/part-r-00000 | head

```
## 10. Building a Webservice to give movie recommendations
Next step in this tutorial is to build a web service to give movie reccomendations to the given user. To do this following libraries are used.
```bash
$ sudo pip3 install twisted
$ sudo pip3 install klein
$ sudo pip3 install redis
```

## 11. Installing and Starting the Redis server
```bash
$ wget http://download.redis.io/releases/redis-2.8.7.tar.gz
$ tar xzf redis-2.8.7.tar.gz
$ cd redis-2.8.7
$ make
$ ./src/redis-server &
```
## 12. Creating the Python Script for movie recommendations
This script will read the hdfs system to see the recommendations for a given user through web service and return the recommendations. The py script in the original tutorial to work correctly some corrections were made and the script is here.
use `$ sudo nano hello.py` to create the .py file and copy the following python script to it.
```python
from klein import run, route
import redis
import os

# Start up a Redis instance
r = redis.StrictRedis(host='localhost', port=6379, db=0)

# Pull out all the recommendations from HDFS
p = os.popen("hadoop fs -cat recommendations/part*")

# Load the recommendations into Redis
for i in p:

  # Split recommendations into key of user id 
  # and value of recommendations
  # E.g., 35^I[2067:5.0,17:5.0,1041:5.0,2068:5.0,2087:5.0,
  #       1036:5.0,900:5.0,1:5.0,081:5.0,3135:5.0]$
  k,v = i.split('\t')

  # Put key, value into Redis
  r.set(k,v)

# Establish an endpoint that takes in user id in the path
@route('/<string:id>')

def recs(request, id):
  # Get recommendations for this user
  v = r.get(id)
  return 'The recommendations for user '+str(id)+' are '+str(v)


# Make a default endpoint
@route('/')

def home(request):
  return 'Please add a user id to the URL, e.g. http://localhost:8081/1234n'

# Start up a listener on port 8081
run("localhost", 8081)

```
You can create a new file with name *hello.py* with command `$ sudo nano hello.py`  and update the text with the above script. Then save it. Run the script with twistd.
```bash
$ twistd -noy hello.py &
```

## Testing the Service
You can simply test the service with 

```bash
$ curl localhost:8081/37 
```
This should give a output like the following which says the movies recommended to the user id 37.
```
The recommendations for user 37 are b'[265:5.0,3168:5.0,2572:5.0,3035:5.0,2375:5.0,2243:5.0,594:5.0,1517:5.0,1188:5.0,1449:5.0]\
```

## Shutting down the Clusters

This is the most important step if you don't won't to loose lot of money ;) . 