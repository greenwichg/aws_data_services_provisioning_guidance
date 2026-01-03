# AWS Cloud Data Engineering End-to-End Project

## AWS Glue ETL Job, S3, Apache Spark

<img src="../images/aws_ingestion_glue/image_1.png" alt="Architecture Diagram" width="600">

[![GitHub Repository](https://img.shields.io/badge/GitHub-Repository-blue?logo=github)](https://github.com/dogukannulu/glue_etl_job_data_catalog_s3)

---

## Tech Stack

- AWS Glue Data Catalog
- AWS Glue Crawler
- AWS Glue ETL Job
- Apache Spark
- Amazon S3
- SQL
- Python

## Overview

In this project, we will first create a new S3 bucket and upload a remote CSV file into that S3 bucket. We are going to create a Data Catalog using either Crawler or a custom schema. Once all is created, we are going to create a new Glue ETL Job. We are going to go through both options: Spark script and Jupyter Notebook. We will do some transformations using Spark. Once our data frame is clear and ready, we will upload it as a parquet file to S3 and will create a corresponding Data Catalog as well. We are going to query the data using Athena and S3 Select. We will also schedule the job so that it runs on a regular basis. 

Think of this project as a production pipeline. Assume that the CSV file located in the S3 is modified at some certain times but the schema stays the same and we should create the corresponding modified parquet file in another directory.

## IAM Roles

First of all, we have to create the necessary policy and role for AWS Glue to be able to watch the logs and connect to S3.

### Create IAM Policy

The first thing's first, we should create a new policy that allows IAM policy for Glue. We can choose the service as Glue and create the policy with the below JSON.

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "VisualEditor0",
            "Effect": "Allow",
            "Action": "iam:PassRole",
            "Resource": "arn:aws:iam::<account_id>:role/*"
        }
    ]
}
```

We can populate the `account_id` part with our own account ID. We should create the policy with a certain name. Then, we can create an IAM role with the above policy. 

### Attach Additional Policies

We should also add the following policies:

- `CloudWatchFullAccessV2`
- `AmazonS3FullAccess`
- `AwsGlueSessionUserRestrictedNotebookPolicy`

We can give a certain name for our newly created role as well.

<img src="../images/aws_ingestion_glue/image_2.png" alt="Architecture Diagram" width="600">

## S3 Bucket

After creating the necessary IAM role, we should create the S3 bucket. This bucket will be used both to get the initial data and upload the final data. If we want to upload the data automatically from inside the EC2 instance, all details can be found in the article: [How to Automate Data Upload to Amazon S3](https://medium.com/@dogukannulu/how-to-automate-data-upload-to-amazon-s3).

If we want to create the S3 bucket manually, we can do it via the S3 dashboard directly and upload the CSV file using AWS CLI. For this article, we will be using a bucket named `aws-glue-etl-job-spark`. This part is important since the bucket name should include "aws-glue", if not we should define some other permissions. We will upload our initial CSV file into this bucket with the key `ufo_reports_source_csv/uforeports.csv`. We are going to use this file as our main source of data.

<img src="../images/aws_ingestion_glue/image_3.png" alt="Architecture Diagram" width="600">

## Glue Crawler / Data Catalog

In this part, the first thing is creating a new database. We can create the database from **AWS Glue** → **Databases** → **Add database**. We can name it `glue-etl-from-csv-to-parquet`.

<img src="../images/aws_ingestion_glue/image_4.png" alt="Architecture Diagram" width="600">

### Create Table

The second part is creating a new table in this database. We are going to use this table to keep the metadata of the object we recently put into the S3 bucket. We can do it both by using Crawler and by editing the schema manually. We will go through editing the schema manually. Our table name will be `ufo_reports_source_csv` (the same as the S3 prefix).

<img src="../images/aws_ingestion_glue/image_5.png" alt="Architecture Diagram" width="600">

We are going to choose the source data as **S3** and browse the location of our newly created object. Be careful that we should choose the directory instead of the file itself at this point.

<img src="../images/aws_ingestion_glue/image_6.png" alt="Architecture Diagram" width="600">

### Define Schema

For our data ufo reports, we should choose the data type as **CSV** and define the schema manually. The main difference between the Crawler and custom schema selection is this point. If we chose Crawler, the schema would be detected automatically but we had to pay some cost. That's the main reason we chose custom schema. If we use Crawler, every option will be the same apart from the schema since it will be determined automatically by Crawler. We can also create a schema registry for the frequently used data catalogs as another option.

<img src="../images/aws_ingestion_glue/image_7.png" alt="Architecture Diagram" width="600">

After creating the source Data Catalog (database and table) we can now create the Glue ETL job.

## Glue ETL Job Using Spark Script

📂 [View Spark script on GitHub](https://github.com/dogukannulu/glue_etl_job_data_catalog_s3/blob/main/glue_jobs/glue_etl_job_spark_script.py)

The main purpose of this Glue ETL job is to modify the source CSV file using the Glue Data Catalog and upload the modified data frame in the parquet format into S3 and create a corresponding target data catalog that keeps the metadata information of the target object.

We are going to create a job using the Spark script editor.

<img src="../images/aws_ingestion_glue/image_8.png" alt="Architecture Diagram" width="600">

### Configure Script

It will lead us to the script page. Before going through the script, we can configure our script. We should first give our script a name and choose the recently created IAM role.

<img src="../images/aws_ingestion_glue/image_9.png" alt="Architecture Diagram" width="600">

For this specific task, choosing the minimum resource power will be enough since the data load is not that big. We will choose the number of workers as **2** and the worker type as **G 1X**.

<img src="../images/aws_ingestion_glue/image_10.png" alt="Architecture Diagram" width="600">

### Import Libraries and Initialize

After configuring our script, we can now create our script. The first part is automatically provided by the Glue itself and it creates a Glue dynamic frame. We are going to add some specific imports and create a Spark data frame out of the Glue dynamic frame.

```python
import sys
from awsglue.transforms import *
from awsglue.utils import getResolvedOptions
from pyspark.context import SparkContext
from awsglue.context import GlueContext
from awsglue.job import Job
from awsglue.dynamicframe import DynamicFrame
from pyspark.sql import functions as F
from pyspark.sql.window import Window

## @params: [JOB_NAME]
args = getResolvedOptions(sys.argv, ['JOB_NAME'])

sc = SparkContext()
glueContext = GlueContext(sc)
spark = glueContext.spark_session
job = Job(glueContext)
job.init(args['JOB_NAME'], args)

glue_dynamic_frame_initial = glueContext.create_dynamic_frame.from_catalog(
    database='glue-etl-from-csv-to-parquet', 
    table_name='ufo_reports_source_csv'
)

df_spark = glue_dynamic_frame_initial.toDF()
```

### Prepare DataFrame

We are going to prepare our Spark data frame for further steps. We will remove NULLs, drop unnecessary columns and rename some of them.

```python
def prepare_dataframe(df):
    """Rename the columns, extract year and drop unnecessary columns. Remove NULL records"""
    df_renamed = df.withColumnRenamed("Shape Reported", "shape_reported") \
        .withColumnRenamed("Colors Reported", "color_reported") \
        .withColumnRenamed("State", "state") \
        .withColumnRenamed("Time", "time")

    df_year_added = df_renamed.withColumn(
        "year", 
        F.year(F.to_timestamp(F.col("time"), "M/d/yyyy H:mm"))
    ) \
        .drop("time") \
        .drop("city")
    
    df_final = df_year_added.filter(
        (F.col("shape_reported").isNotNull()) & 
        (F.col("color_reported").isNotNull())
    )
    
    return df_final
```

### Join DataFrames

Using these transformations, we aim to obtain a data frame that will show us the most occurred color and shapes per year and state breakdown. That's why we will create two separate data frames and join them for this purpose.

```python
def join_dataframes(df):
    """Create color and shape dataframes and join them"""
    shape_grouped = df.groupBy("year", "state", "shape_reported") \
        .agg(F.count("*").alias("shape_occurrence"))

    color_grouped = df.groupBy("year", "state", "color_reported") \
        .agg(F.count("*").alias("color_occurrence"))

    df_joined = shape_grouped.join(
        color_grouped,
        on=["year", "state"],
        how="inner"
    )
    
    return df_joined
```

### Create Final DataFrame

We are now going to get only the maximum number of occurrences using the `row_number` function. We are also familiar with this function from the SQL queries, it will give us the maximum occurrences partitioned by year and state.

```python
def create_final_dataframe(df):
    """Create final dataframe"""
    shape_window_spec = Window.partitionBy("year", "state").orderBy(
        F.col("shape_occurrence").desc()
    )
    color_window_spec = Window.partitionBy("year", "state").orderBy(
        F.col("color_occurrence").desc()
    )

    # Selecting top occurrences of shape and color per year and state
    final_df = df.withColumn("shape_rank", F.row_number().over(shape_window_spec)) \
        .withColumn("color_rank", F.row_number().over(color_window_spec)) \
        .filter((F.col("shape_rank") == 1) & (F.col("color_rank") == 1)) \
        .select(
            "year", "state", "shape_reported", "shape_occurrence", 
            "color_reported", "color_occurrence"
        ) \
        .orderBy(F.col("shape_occurrence").desc())
    
    return final_df
```

### Write to S3 and Data Catalog

Now we will convert our Spark data frame back into a Glue dynamic frame. This will be used for writing the data both to the S3 bucket and creating a corresponding table in the Data Catalog.

```python
df_prepared = prepare_dataframe(df_spark)
df_joined = join_dataframes(df_prepared)
df_final = create_final_dataframe(df_joined)

# From Spark dataframe to glue dynamic frame
glue_dynamic_frame_final = DynamicFrame.fromDF(df_final, glueContext, "glue_etl")

# Write the data in the DynamicFrame to a location in Amazon S3 
# and a table for it in the AWS Glue Data Catalog
s3output = glueContext.getSink(
    path="s3://aws-glue-etl-job-spark/ufo_reports_target_parquet",
    connection_type="s3",
    updateBehavior="UPDATE_IN_DATABASE",
    partitionKeys=[],
    compression="snappy",
    enableUpdateCatalog=True,
    transformation_ctx="s3output",
)

s3output.setCatalogInfo(
    catalogDatabase="glue-etl-from-csv-to-parquet", 
    catalogTableName="ufo_reports_target_parquet"
)

s3output.setFormat("glueparquet")
s3output.writeFrame(glue_dynamic_frame_final)

job.commit()
```

After populating the script page as above, we can now **Save** the script and **Run** it. We can monitor the script from the **Runs** tab. We can wait until the run succeeds.

## Glue ETL Job Using Jupyter Notebook

📂 [View Jupyter Notebook on GitHub](https://github.com/dogukannulu/glue_etl_job_data_catalog_s3/blob/main/glue_jobs/glue_etl_job_spark_notebook.ipynb)

One of the other options is creating the ETL job using a Jupyter Notebook. The steps are pretty similar to the Spark script. Therefore, there's no need to go through the Jupyter Notebook code itself. 

### Differences Between Spark Script and Jupyter Notebook

- We should choose **Jupyter Notebook** instead of Spark script in the beginning while creating the ETL job
- We should define the resource power in the notebook itself instead of configuring it

```python
%idle_timeout 2880
%glue_version 3.0
%worker_type G.1X
%number_of_workers 2
```

All the remaining parts will be the same. You might take a look at the above Jupyter notebook for further details.

## Monitor Data Using AWS Athena and S3 Select

After the run is complete and it succeeds, we can first check the S3 bucket. A new directory should be created inside the bucket named `ufo_reports_target_parquet` and the resulting parquet file should be loaded into it.

<img src="../images/aws_ingestion_glue/image_12.png" alt="Architecture Diagram" width="600">

<img src="../images/aws_ingestion_glue/image_13.png" alt="Architecture Diagram" width="600">

### S3 Select

We can click on the parquet file and choose **S3 Select** from the actions. We can run a simple S3 query and see the first 5 rows for example.

### Check Glue Data Catalog

After checking the data in the S3 bucket, we can also check if the table is created in the Glue Data Catalog.

<img src="../images/aws_ingestion_glue/image_14.png" alt="Architecture Diagram" width="600">

We can see that `ufo_reports_target_parquet` table is created in Glue Data Catalog. We can also see that the database is the one we recently created. The main location of this table is the parquet S3 object under `aws-glue-etl-job-spark/ufo_reports_target_parquet/`. This table represents the metadata of that object.

### Query with Athena

We can now click on the table and choose **View data** from Actions.

<img src="../images/aws_ingestion_glue/image_15.png" alt="Architecture Diagram" width="600">

This will lead us through Amazon Athena. We can run the below simple SQL query to see the first 10 rows of the data.

```sql
SELECT * 
FROM "AwsDataCatalog"."glue-etl-from-csv-to-parquet"."ufo_reports_target_parquet" 
LIMIT 10;
```

We can see the resulting data below:

- **Final Parquet File**
- **Final CSV File**

### Best Practices for Athena Queries

We can run SQL queries in Athena. We should be careful about the below-mentioned points:

- We should use `LIMIT` if we want to see a certain part of the data
- We can use some window functions and aggregations to restrict the data
- We can also choose the columns we want to see
- We should avoid scanning the whole table by adding a `WHERE` clause if suitable for our use case
- Overall, we should be careful about restricting the query so that it only shows the target data. These will help us avoid extra costs.

## Scheduling the Glue ETL Job

We can add a schedule to our newly created Glue ETL job. If we want to run the job on a regular basis, we should add a suitable schedule using **Cron**. We may see the below schedule which will be running at 10 AM UTC every day.

<img src="../images/aws_ingestion_glue/image_16.png" alt="Architecture Diagram" width="600">

**Example Cron Expression:**
```
0 10 * * ? *
```

This schedule will run the job at 10:00 AM UTC every day.

---
