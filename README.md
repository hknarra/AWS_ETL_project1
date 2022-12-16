# AWS ETL project (S3 bucket to Redshift):
Once the file is uploaded in S3 bucket source key location, S3 bucket event invokes the Lambda function.
This Lambda function triggers the glue job. 
Glue job fetches the source data files from s3 key bucket location using the meta data schema in Glue Catalog and does transformations and loads into the Database tables in Redshift.

Create a crawler and crawl the file for the 1st time, enable versioning on the bucket.
Same file name lands in S3 key location every week.

### 1) S3 bucket creation:
Services Storage S3 Buckets create a bucket: bucket name (data1) 
region (select) create bucket


### 2) create VPC endpoint:
Create endpoint for s3
Take data from S3 and load into Redshift. Bridge for this to happen is VPC endpoint.
Create endpoint in VPC:
VPC Endpoints create endpoint name (vpc-endpoint-s3) aws services 
select S3 (Gateway) select VPC, route table 
Policy: Full access Create endpoint

You get error if VPC s3 endpoint is not created.


### 3) create security group s3 to redshift:
Setting up vpc to connect to JDBC:
Add a security group with 5439 port open to default SG:
VPC Security Groups Default security group Edit inbound rules
Type (redshift), 5439, your-ip/32 –inbound rules

Ip4.me to check Ip of your computer


### 4) create glue crawler:
To crawl source in S3.
Add classifier:
Glue classifier (under data catalog) add classifier give details create 

Create crawler:
Partition Tables:
Where we want to see active data only. Old data rarely.
Glue tables add tables (using crawler) name (crawler1), 
 Data stores S3, include path (s3://data1/
next …create crawler

run crawler
Preview data and see
2 tables created 1 for old, 1 for active.


### 5) Create Redshift cluster:
#### Create Role:
Role: AWS service redshift Redshift customizable Next 
AmazonRedshiftFullaccess, DMSRedShiftS3Role, S3FullAccess 
 Name (redshift-full-access) create Copy ARN

#### Create Redshift Spectrum Cluster:
Not free: 
Redshift Clusters create cluster  cluster identifier (redshift-cluster-hk-1),
Production, dc2.large Nodes (1) 
Master user name (awsuser), password (Awsuser1), port (5439)
cluster Permissions: Role (select above role) 
additional configurations: use defaults (disable)
network & security (publicly accessible (enable))
create cluster
select above role.
Takes 5-10 min to become available.

Attach Iam role:
Cluster Actions Manage Iam roles: (select redshift-full-access) 
associate Iam role save changes

Create table in Dev db:
create table city_data(
    id bigint not null distkey sortkey,
    country varchar(100),
    state varchar(50),
    City varchar(100),
    amount decimal(5,2)
    );



### 6) Create JDBC connection in glue:
Create connection:
Glue studio connectors create connection
name (Redshift), connection type (Amazon Redshift),
Connection access: Db instance (select cluster), credentials 
select Network Options
create connection

### 7) Create crawler to crawl target:
Create crawler:
Create crawler to crawler target and generate schema table.
Name ( ) Data sources (add a data source (Redshift), connection (give above connection))
Include path (dev/public/tablename)
Role(my-glue-s3-access)


### 8) create Glue job:
Fetch data file from s3 via glue Catalog table and load into Redshift
```
import sys
from awsglue.transforms import *
from awsglue.utils import getResolvedOptions
from pyspark.context import SparkContext
from awsglue.context import GlueContext
from awsglue.job import Job

args = getResolvedOptions(sys.argv, ["JOB_NAME"])
sc = SparkContext()
glueContext = GlueContext(sc)
spark = glueContext.spark_session
job = Job(glueContext)
job.init(args["JOB_NAME"], args)

# Script generated for node S3 bucket
S3bucket_node1 = glueContext.create_dynamic_frame.from_catalog(
    database="glue_db1", table_name="data1", transformation_ctx="S3bucket_node1"
)

# Script generated for node ApplyMapping
ApplyMapping_node2 = ApplyMapping.apply(
    frame=S3bucket_node1,
    mappings=[
        ("id", "long", "id", "long"),
        ("country", "string", "country", "string"),
        ("state", "string", "state", "string"),
        ("city", "string", "city", "string"),
        ("amount", "double", "amount", "double"),
    ],
    transformation_ctx="ApplyMapping_node2",
)

# Script generated for node Redshift Cluster
RedshiftCluster_node3 = glueContext.write_dynamic_frame.from_catalog(
    frame=ApplyMapping_node2,
    database="glue_db1",
    table_name="dev_public_city_data",
    redshift_tmp_dir="s3://glue-output/glue_job_temp/",
    transformation_ctx="RedshiftCluster_node3",
)

job.commit()
```


### 9) Create lambda function to invoke Glue Job:
** Create Policy:**
Iam Access management policies 
create policy Choose service (Glue)
Actions (write StartJobRun)
Resources: (specific, give details add) 
Next Name (Policy_Glue_Job_run) create 

Role for Lambda function to access Glue Job:
Add above policy to role: Lambda_glue_service_role
If above policy does not work, choose GlueServiceRolePolicy.

**Lambda function: function_strt_Glue_job **
```
# Set up logging
import json
import os
import logging
logger = logging.getLogger()
logger.setLevel(logging.INFO)

# Import Boto 3 for AWS Glue
import boto3
client = boto3.client('glue')

# Variables for the job: 
glueJobName = "s3-red-new"

# Define Lambda function
def lambda_handler(event, context):
    logger.info('## INITIATED BY EVENT: ')
    #logger.info(event['detail'])
    response = client.start_job_run(JobName = glueJobName)
    logger.info('## STARTED GLUE JOB: ' + glueJobName)
    logger.info('## GLUE JOB RUN ID: ' + response['JobRunId'])
    return response
```

### 10) S3 event to invoke lambda function:
Enable versioning in S3 bucket.

S3 event notification:
select bucket properties
event notifications: create event notification name (invokeLambda_S3file), Event types (select all), Destination (lambda) select lambda function
save changes
