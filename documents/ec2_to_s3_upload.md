# How to Automate Data Upload to Amazon S3

<img src="../images/ec2_to_s3/image_1.png" alt="Architecture Diagram" width="600">

[![GitHub Repository](https://img.shields.io/badge/GitHub-Repository-blue?logo=github)](https://github.com/greenwichg/send_data_to_aws_services/tree/main/csv_to_s3)

---

## Tech Stack

- Python
- Shell Scripting
- Amazon S3
- Amazon EC2

## Overview

- We will be running all the scripts inside the EC2 instance. So, we have to first create a new EC2 instance with suitable IAM roles.

- Then we are going to create a Python script that gets a CSV file from a URL. This script will also create a new S3 bucket and upload the retrieved CSV file into the S3 bucket.

- We are going to automate the whole process using a shell script and will be able to monitor live logs.

- This practice might be used to trigger the Lambda function or any other operation that we would like to handle from inside the EC2.

## IAM Role

Since we want to create an S3 bucket using the Python script and upload data into it inside an EC2 instance, we have to create a proper IAM role to access S3 from inside the EC2.

First of all from **IAM** → **Roles** → **Create role**

<img src="../images/ec2_to_s3/image_2.png" alt="Architecture Diagram" width="600">

This will allow us to do the desired actions inside the EC2 instance. After clicking **Next**, we can choose **AmazonS3FullAccess**. This is not the best practice and we can limit the privileges using a JSON. But for this specific use case, we will use this access role.

After giving a proper name for our role (for example `ec2-s3-full-access`), we can create the role. We will later need this name while attaching the IAM role for our instance.

## EC2 Instance

We should create a **Key Pair** before creating the EC2 instance and install .pem file onto our local machine.

We should also create a **Security Group** from the left panel on the EC2 dashboard. Since we will need SSH to connect to the instance from our local machine, we have to define SSH with all IPs allowed (not the best practice, we can also choose only our IP).

Once the security group and key pair are created, we can now create the EC2 instance. We can click on **Launch instance** and set the configurations as below:

- **Application and OS Images** → Amazon Linux (Free tier eligible)
- **Instance type** → t2.micro (free tier eligible)
- **Key pair** → We can choose the key pair we created and install .pem file to our local machine
- **Network settings** → Select the existing security group and we can select the one we created
- We should also attach the previously created IAM role `ec2-s3-full-access` to our instance
- We can leave other fields as default and launch the instance.

## Python Script

We will be using [this CSV file](https://github.com/greenwichg/send_data_to_aws_services/blob/main/csv_to_s3/dirty_store_transactions.csv) as an example for the whole process. If you want to use any other URL for the CSV file, you can modify the shell script which will be explained in the next section. You can also see the [requirements.txt](https://github.com/dogukannulu/send_data_to_aws_services/blob/main/csv_to_s3/requirements.txt) file which will be automatically installed into the EC2 instance via shell script later.

### Import Libraries and Setup Logger

We have to first import the necessary libraries and define the logger. Logger will be important since we will be monitoring the logs of the script.

```python
import boto3
import requests
import logging
import argparse
from botocore.exceptions import ClientError

logging.basicConfig(level=logging.INFO,
                    format='%(asctime)s - %(name)s - %(levelname)s - %(message)s')
logger = logging.getLogger(__name__)
```

### S3Uploader Class

We are going to create a class where we handle all S3-related processes. `create_bucket` function will create an S3 bucket with the desired name. We should be careful with the naming since it has to be unique and there are some rules for the naming. You can see the [rules for the bucket naming here](https://docs.aws.amazon.com/AmazonS3/latest/userguide/bucketnamingrules.html). As you can also see, the script is valid for `eu-central-1` region. We can modify that part if our bucket is located in another region. `put_object` will put the CSV file directly into the S3 bucket without doing any modification. Since this script is valid for all remote CSV files, I didn't add any modification parts. If necessary, you might add the modification part as well.

```python
class S3Uploader:
    def __init__(self, region='eu-central-1'):
        self.s3_client = boto3.client('s3')
        self.region = region

    def create_bucket(self, name):
        """
        Create an S3 bucket with the specified region and name
        """
        try:
            self.s3_client.create_bucket(
                Bucket=name,
                CreateBucketConfiguration={
                    'LocationConstraint': self.region
                }
            )
            logger.info(f"Bucket '{name}' created successfully.")
        except ClientError as e:
            if e.response['Error']['Code'] == 'BucketAlreadyOwnedByYou':
                logger.warning(f"Bucket '{name}' already exists and is owned by you.")
            elif e.response['Error']['Code'] == 'BucketAlreadyExists':
                logger.warning(f"Bucket '{name}' already exists and is owned by someone else.")
            else:
                logger.warning("An error occurred:", e)

    def put_object(self, bucket_name, object_key, csv_data):
        """
        Upload the CSV data into the specified S3 bucket
        """
        try:
            self.s3_client.put_object(Body=csv_data, Bucket=bucket_name, Key=object_key)
            logger.info('Object uploaded into S3 successfully')
        except ClientError as e:
            logger.warning("An error occurred while putting the object into S3:", e)
        except Exception as e:
            logger.warning("An unexpected error occurred while putting the object into S3:", e)
```

### Define Command-Line Arguments

We are now going to define the command line arguments.

**Arguments:**
- **bucket_name** (str): We should be careful with the naming as mentioned before
- **object_key** (str): The key of the object. The exact location of the CSV file in the bucket in other words
- **data_url** (str): URL of the CSV file. I will be using my Github URL, but you can use any other URL if valid

```python
def define_arguments():
    """
    Defines the command-line arguments
    """
    parser = argparse.ArgumentParser(description="Upload CSV data to Amazon S3")
    parser.add_argument("--bucket_name", "-bn", required=True, help="Name of the S3 bucket")
    parser.add_argument("--object_key", "-ok", required=True, help="Name for the S3 object")
    parser.add_argument("--data_url", "-du", required=True, help="URL of the remote CSV file")
    args = parser.parse_args()

    return args
```

### Main Function

In the end, we can combine all of them.

```python
def main():
    args = define_arguments()
    csv_data = requests.get(args.data_url).content
    s3_uploader = S3Uploader()
    s3_uploader.create_bucket(name=args.bucket_name)
    s3_uploader.put_object(bucket_name=args.bucket_name, object_key=args.object_key, csv_data=csv_data)
```

## Shell Script

The first part was creating the Python script, now we are going to automate the running process with a shell script. First of all, we are going to define the location of the log file and the structure of the logs.

```bash
#!/bin/bash

log_file="/project/upload_csv_to_s3.log"

# Function to log messages to a log file
log_message() {
    local log_text="$1"
    echo "$(date +'%Y-%m-%d %H:%M:%S') - $log_text" >> "$log_file"
}
```

We can monitor all the logs in `/project/upload_csv_to_s3.log`. Now we have to check the root privileges since we have to run all commands with root privileges without using sudo.

```bash
# Function to check if the script is being run with root privileges
check_root_privileges() {
    if [[ $EUID -ne 0 ]]; then
        log_message "This script is running with root privileges"
        exit 1
    fi
}
```

### Install Required Packages

To prepare the EC2 instance for all the processes, we have to first install the necessary packages, modules, and libraries:

- python3
- pip3
- wget
- unzip

```bash
# Function to install packages using yum
install_packages() {
    local packages=(python3 python3-pip wget unzip)
    log_message "Installing required packages: ${packages[*]}"
    yum update -y
    yum install -y "${packages[@]}"
}
```

### Download and Unzip Files

We are going to start the main process after getting the machine ready. The first thing is downloading the zip file from the above Github repo. This [zip file](https://github.com/greenwichg/send_data_to_aws_services/blob/main/csv_to_s3/csv_to_s3.zip) includes the [Python script](https://github.com/dogukannulu/send_data_to_aws_services/blob/main/csv_to_s3/csv_to_s3.py) and [requirements.txt](https://github.com/greenwichg/send_data_to_aws_services/blob/main/csv_to_s3/requirements.txt). Then, we will unzip the downloaded zip file.

```bash
# Function to download the zip file
download_zip_file() {
    local project_dir="/project"
    log_message "Downloading the zip file"
    cd "$project_dir" || exit 1
    wget -q https://github.com/dogukannulu/send_data_to_aws_services/raw/main/csv_to_s3/csv_to_s3.zip
}

# Function to unzip the files
unzip_files() {
    log_message "Unzipping the files"
    unzip -o csv_to_s3.zip
}
```

### Install Python Libraries

To run our Python script, we have to have all the libraries and packages installed in the instance. Therefore, we are going to install all of them if not exist.

```bash
# Function to install required Python libraries
install_python_libraries() {
    local requirements_file="requirements.txt"
    log_message "Installing Python libraries from $requirements_file"
    pip3 install -r "$requirements_file"
}
```

### Execute Python Script

Here comes the main part. We are going to define all the command line arguments in the below function. Then, we will run the Python script with those arguments. *This part might be modified if necessary*.

```bash
# Function to execute the Python script
execute_python_script() {
    local csv_to_s3_script="csv_to_s3.py"
    local bucket_name="csv-to-s3-project-dogukan-ulu"
    local object_key="dirty_store_transactions/dirty_store_transactions.csv"
    local data_url="https://raw.githubusercontent.com/dogukannulu/send_data_to_aws_services/main/csv_to_s3/dirty_store_transactions.csv"
    
    log_message "Executing the Python script"
    chmod +x "$csv_to_s3_script"
    python3 "$csv_to_s3_script" --bucket_name "$bucket_name" \
        --object_key "$object_key" \
        --data_url "$data_url"
}
```

### Main Function

In the end, we are going to combine all these and log all the stdout and stderr into the log file.

```bash
# Main function to run the entire script
main() {
    log_message "Starting the script"
    check_root_privileges
    install_packages
    download_zip_file
    unzip_files
    install_python_libraries
    execute_python_script
    log_message "Script execution completed"
}

# Run the main function and redirect stdout and stderr to the log file
main >> "$log_file" 2>&1
```

## Automate the Process

We have understood how Python and shell scripts exactly work. Here comes the only part that we will be running inside the EC2 instance. First of all, we have to be located in the root directory. All the processes will be in `/project` directory. We will download the shell script in the root directory (we can also download it into the `/project`).

If any modification is necessary for the shell script, we can run `sudo vi setup.sh`. If not, we are only going to run the below commands:

```bash
sudo curl -O https://raw.githubusercontent.com/dogukannulu/send_data_to_aws_services/main/csv_to_s3/setup.sh
sudo chmod +x setup.sh
sudo mkdir project
sudo ./setup.sh
sudo tail -f /project/upload_csv_to_s3.log
```

### What These Commands Do

1. Download the `setup.sh` into the EC2 instance
2. Make `setup.sh` executable
3. Create the directory `/project`
4. Execute `setup.sh`
5. Monitor the live logs (This might require opening another terminal window)

In the end, you can check if the CSV file is uploaded into the S3 bucket.

---
