# How to Create Amazon MSK Cluster and Stream to Kafka within EC2

<img src="../images/ec2_to_msk/image_1.png" alt="Architecture Diagram" width="600">

---

## Tech Stack

- Amazon MSK
- Apache Kafka
- Amazon EC2

## Overview

In this article, we are going to create both serverless and provisioned Amazon MSK clusters. Then, we will create an EC2 instance where we will be running our consumer and producer as bash commands. We are going to create bash commands which will create a new topic, send the messages we will be writing line by line to this topic (producer) and consume the messages on the console.

For the streaming data, Amazon Kinesis can also be used but in this example, we will be using serverless and provisioned clusters instead. The advantage of using serverless over provisioned is running clusters without having to manage compute and storage capacity.

## Amazon MSK Serverless Cluster

Since Amazon MSK is a comparably higher-cost AWS service, we are going to configure the cluster with the minimum requirements possible. After going to the MSK dashboard and clicking **Create Cluster** button, we can follow the below instructions:

<img src="../images/ec2_to_msk/image_2.png" alt="Architecture Diagram" width="600">

- **Creation method:** Quick create
- **Cluster name:** kafka-msk-cluster
- **Cluster type:** Serverless

From the table under **All cluster settings**, copy the values of the following settings and save them since we will need them later:

- **VPC:** vpc-...
- **All subnet names**
- **Security groups associated with VPC:** sg-…

Now, we can create the cluster. It will take more than 15 minutes.

## Amazon MSK Provisioned Cluster

After going to the MSK dashboard and clicking **Create Cluster** button, we can follow the below instructions:

- **Creation Method:** Quick Create
- **Cluster Name:** kafka-msk-cluster
- **Cluster Type:** Provisioned
- **Apache Kafka Version:** 3.4.0
- **Broker Type:** kafka.t3.small
- **Amazon EBS Storage per Broker:** 2 GiB

From the table under **All cluster settings**, copy the values of the following settings and save them since we will need them later:

- **VPC:** vpc-...
- **All subnet names**
- **Security groups associated with VPC:** sg-…

Now, we can create the cluster. It will take more than 15 minutes.

## Create IAM Policy

Choosing `FullMSKAccess` is an option while attaching the IAM role. But restricting the access only to the cluster we created will be a better practice. For this purpose, we are going to go to **IAM** → **Policies** → **Create Policy**. After going to the JSON section, we can define the JSON below:

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "kafka-cluster:Connect",
                "kafka-cluster:AlterCluster",
                "kafka-cluster:DescribeCluster"
            ],
            "Resource": [
                "arn:aws:kafka:eu-central-1:<Account-ID>:cluster/kafka-msk-cluster/*"
            ]
        },
        {
            "Effect": "Allow",
            "Action": [
                "kafka-cluster:*Topic*",
                "kafka-cluster:WriteData",
                "kafka-cluster:ReadData"
            ],
            "Resource": [
                "arn:aws:kafka:eu-central-1:<Account-ID>:topic/kafka-msk-cluster/*"
            ]
        },
        {
            "Effect": "Allow",
            "Action": [
                "kafka-cluster:AlterGroup",
                "kafka-cluster:DescribeGroup"
            ],
            "Resource": [
                "arn:aws:kafka:eu-central-1:<Account-ID>:group/kafka-msk-cluster/*"
            ]
        }
    ]
}
```

We can replace `eu-central-1` with the region we are currently using. We have to put our Account ID into the string. We can find our account ID at the top right corner, from the account information.

We can name our policy as `msk-cluster-only-policy`.

<img src="../images/ec2_to_msk/image_3.png" alt="Architecture Diagram" width="600">

## Create IAM Role

The next step is defining the IAM role. The role we will create is going to be used to attach to the EC2 instance we will create. In the end, we will be able to access to the MSK cluster and process the remaining steps.

**IAM** → **Roles** → **Create Role** → Choose common use cases as **EC2**

<img src="../images/ec2_to_msk/image_4.png" alt="Architecture Diagram" width="600">

After going to permissions, we can search for the policy we lately created (`msk-cluster-only-policy`). After all, we can create the role and name it `msk-cluster-only-role`.

## Create EC2 Instance

Once we go to the EC2 main dashboard and click on **Create Instance**, we can follow the next steps:

- **Name** → msk_ec2_instance
- **Application and OS Images** → Amazon Linux (Free tier eligible)
- **Instance type** → t2.micro (free tier eligible)
- **Key pair** → We can choose the key pair and install .pem file to our local machine
- **Network settings** → Select a suitable security group that includes SSH as the inbound rule
- Attach the IAM role we created (`msk-cluster-only-role`) to the instance while creating. Or you can also click on **Actions** → **Security** → **Modify the IAM role** and select the IAM role we created

<img src="../images/ec2_to_msk/image_5.png" alt="Architecture Diagram" width="600">

- We can leave other fields as default and launch the instance.

### Configure Security Groups

After creating the EC2 instance, we know which security group belongs to our instance. We should copy the ID of the security group (let's call it `sg-123`). In the first section, I mentioned that we should remember the security group ID of our MSK cluster (let's call it `sg-456`). We should go into that security group and edit inbound rules. We should define an inbound rule that allows all traffic, select **Custom** as the source, and paste the `sg-123` into the box next to that.

Up to this point, we attached the IAM role we created to our instance so that the instance can access our MSK cluster. We also allowed all traffic coming from the EC2 instance to the MSK cluster by defining security groups. Now, we are going to get our instance ready to produce and consume messages.

## Setup EC2 Instance for Kafka

We have to connect to the EC2 instance with an SSH connection first. You can have a look at my other articles on connecting to the EC2 instance.

### Install Java

First of all, we have to install Java on the instance.

```bash
sudo yum -y install java-11
```

### Download and Extract Kafka

Then, we have to download Kafka binaries and untar it. (The MSK version here is 3.4.0, but you can modify that part according to the version you created the cluster.)

```bash
wget https://archive.apache.org/dist/kafka/3.4.0/kafka_2.13-3.4.0.tgz
tar -xzf kafka_2.13-3.4.0.tgz
```

### Download AWS MSK IAM Auth JAR

We have to download the necessary JAR file as well.

```bash
cd kafka_2.13-3.4.0/libs
wget https://github.com/aws/aws-msk-iam-auth/releases/download/v1.1.1/aws-msk-iam-auth-1.1.1-all.jar
```

### Configure Client Properties

We have to define some configuration parameters so that we can allow the necessary protocols for the later processes. (Assuming we are currently in the directory `~/kafka_2.13-3.4.0/libs`)

```bash
cd ../bin
vi client.properties
```

Populate the file `client.properties` with the following parameters:

```properties
security.protocol=SASL_SSL
sasl.mechanism=AWS_MSK_IAM
sasl.jaas.config=software.amazon.msk.auth.iam.IAMLoginModule required;
sasl.client.callback.handler.class=software.amazon.msk.auth.iam.IAMClientCallbackHandler
```

### Get Bootstrap Server String

We have configured our instance for all the necessary processes. Last but not least, we have to learn `BootstrapServerString` from:

**Cluster Name** → **View Client Information**

We have to note it down since we will need it while writing the commands.

## Create Kafka Topic and Produce Messages

Here comes the main target of the project. We have to first create the Kafka topic, then we will stream messages to the Kafka producer. We have to ensure that we are in the correct directory (`~/kafka_2.13-3.4.0/bin/`). Because all shell scripts are located in this directory.

### Create Kafka Topic

The first command to create a Kafka topic named `msk-kafka-topic` with 3 replicas and 5 partitions:

```bash
./kafka-topics.sh --create --bootstrap-server <BootstrapServerString> \
    --command-config client.properties \
    --replication-factor 3 --partitions 5 \
    --topic msk-kafka-topic
```

We should get a message: **Created topic msk-kafka-topic.**

### Start Producer

After creating the topic, we can now start producing messages manually.

```bash
./kafka-console-producer.sh --broker-list <BootstrapServerString> \
    --producer.config client.properties \
    --topic msk-kafka-topic
```

<img src="../images/ec2_to_msk/image_6.png" alt="Architecture Diagram" width="600">

We can send whatever message we would like. I am sending `message-1`, `message-2`, etc.

## Consume Kafka Messages

Since we already created the topic and produced the custom messages, we can open another shell window and create the Kafka consumer on that one as well. We should connect to the instance again and ensure that we are in the correct directory first. Then, we can run the following command.

```bash
./kafka-console-consumer.sh --bootstrap-server <BootstrapServerString> \
    --consumer.config client.properties \
    --topic msk-kafka-topic
```

<img src="../images/ec2_to_msk/image_7.png" alt="Architecture Diagram" width="600">

We can see that all the messages are obtained on the consumer window. This is just to illustrate how an MSK topic works.

## Important Note

**Please don't forget to delete MSK cluster and EC2 instance to avoid additional cloud costs.**

---
