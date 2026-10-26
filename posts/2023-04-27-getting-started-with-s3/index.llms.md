**Boto3** is an AWS python SDK that allows access to AWS services like EC2 and S3. It provides a python object-oriented API and as well as low-level access to AWS services

**Note:** This post uses minIO to simulate S3 locally so you don’t incur AWS costs. If you’re using real AWS, remove the `endpoint_url` parameter and ensure your credentials are configured.

## Start MinIO docker instance

``` bash
docker run -d --rm -p 9000:9000 -p 9001:9001 \
  -e MINIO_ROOT_USER=minioadmin \
  -e MINIO_ROOT_PASSWORD=minioadmin \
  minio/minio server /data --console-address ":9001"
```

``` bash
pip install boto3
```

``` python
import boto3, botocore
import glob

files = glob.glob("data/*")  # to upload multiple files
files
```

    ['data/Player Data.xlsx',
     'data/30-days-create-folds.ipynb',
     'data/ARK_GENOMIC_REVOLUTION_ETF_ARKG_HOLDINGS.csv',
     'data/star_pattern_turtlesim.png']

## Create a session and client

Boto3’s region defaults to N-Virginia. To create buckets in another region, region name has to be explicitly mentioned using session object.

``` python
session = boto3.Session(
    region_name="us-east-2",
    aws_access_key_id="minioadmin",
    aws_secret_access_key="minioadmin",
)
s3client = session.client(
    "s3",
    endpoint_url="http://localhost:9000",
    config=boto3.session.Config(signature_version="s3v4"),
)
s3resource = session.resource(
    "s3",
    endpoint_url="http://localhost:9000",
    config=boto3.session.Config(signature_version="s3v4"),
)
```

S3 buckets have to follow bucket naming [rules](https://docs.aws.amazon.com/AmazonS3/latest/userguide/bucketnamingrules.html).

``` python
bucket_names = ['my-s3bucket1-usohio-region', 'my-s3bucket2-usohio-region']
s3location = {'LocationConstraint': 'us-east-2'}
```

## Check if bucket exists in S3

Checking for something before creation is one of the important tasks to avoid unnecessary errors. Here we check if the buckets already exists.

``` python
def check_bucket(bucket):
    """
    Checks if a bucket is present in S3
    args:
    bucket: takes bucket name
    """
    try:
        s3client.head_bucket(Bucket=bucket)
        print('Bucket exists')
        return True
    except botocore.exceptions.ClientError as e:
        # If a client error is thrown, then check that it was a 404 error.
        # If it was a 404 error, then the bucket does not exist.
        error_code = int(e.response['Error']['Code'])
        if error_code == 403:
            print("Private Bucket. Forbidden Access!")
            return True
        elif error_code == 404:
            print("Bucket Does Not Exist!")
            return False
```

``` python
for bucket in bucket_names: 
    print(check_bucket(bucket))
```

    Bucket Does Not Exist!
    False
    Bucket Does Not Exist!
    False

## Create a bucket in S3

If the buckets don’t exist, we create them. We need to supply bucket name, a dictionary specifying in which region the bucket has to be created.

``` python
for bucket_name in bucket_names:
    if not (check_bucket(bucket_name)):
        print("Creating a bucket..")
        s3client.create_bucket(Bucket=bucket_name, CreateBucketConfiguration=s3location)
```

    Bucket Does Not Exist!
    Creating a bucket..
    Bucket Does Not Exist!
    Creating a bucket..

## Bucket Versioning

Bucket versioning initial state is not set by default. The response from when not initialised doesn’t carry status information rather status dict is absent. Status expects two return states: **enabled**, **suspended**. On first creation, the status is in disabled, an unknown state.

So in order to make it appear in the REST response, bucket must be enabled by calling the `BucketVersioning()` boto3 resource function. If we then check the status, it will be present in the REST response.

``` python
def get_buckets_versioning_client(bucketname):
    """
    Checks if bucket versioning is enabled/suspended or initialised
    Args:
    bucketname: bucket name to check versioning
    Returns: response status - enabled or suspended
    """
    response = s3client.get_bucket_versioning(Bucket=bucketname)
    if "Status" in response and (
        response["Status"] == "Enabled" or response["Status"] == "Suspended"
    ):
        print(f"Bucket {bucketname} status: {response['Status']}")
        return response["Status"]
    else:
        print(
            f"Bucket versioning not initialised for bucket: {bucketname}. Enabling..."
        )
        s3resource.BucketVersioning(bucket_name=bucketname).enable()
        enable_response = s3resource.BucketVersioning(bucket_name=bucketname).status
        return enable_response
```

``` python
for bucket_name in bucket_names: 
    version_status = get_buckets_versioning_client(bucket_name)
    print(f'Versioning status: {version_status}')
```

    Bucket versioning not initialised for bucket: my-s3bucket1-usohio-region. Enabling...
    Versioning status: Enabled
    Bucket versioning not initialised for bucket: my-s3bucket2-usohio-region. Enabling...
    Versioning status: Enabled

## To suspend bucket versioning

``` python
for bucket_name in bucket_names:
    version_status = get_buckets_versioning_client(bucket_name)
    print(f'Versioning status: {version_status}')
    if version_status == 'Enabled':
        print('Disabling again..')
        s3resource.BucketVersioning(bucket_name=bucket_name).suspend()
```

    Bucket my-s3bucket1-usohio-region status: Enabled
    Versioning status: Enabled
    Disabling again..
    Bucket my-s3bucket2-usohio-region status: Enabled
    Versioning status: Enabled
    Disabling again..

## To enable bucket versioning

``` python
for bucket_name in bucket_names:
    version_status = get_buckets_versioning_client(bucket_name)
    print(f'Versioning status: {version_status}')
    if version_status == 'Suspended':
        print('Enabling again..')
        s3resource.BucketVersioning(bucket_name=bucket_name).enable()
```

    Bucket my-s3bucket1-usohio-region status: Suspended
    Versioning status: Suspended
    Enabling again..
    Bucket my-s3bucket2-usohio-region status: Suspended
    Versioning status: Suspended
    Enabling again..

## Get bucket list from S3

We can list the buckets in S3 using `list_buckets()` client function. It return a dict. We can iterate through `Buckets` key to find the names of the buckets.

``` python
buckets_list = s3client.list_buckets()
for bucket in buckets_list['Buckets']:
    print(bucket['Name'])
```

    my-s3bucket1-usohio-region
    my-s3bucket2-usohio-region

## Upload files to S3

Boto3 allows file upload to S3. The `upload_file` client function requires three mandatory arguments -

    1. filename of the file to be uploaded
    2. bucket_name, Into which bucket the file would be uploaded
    3. key, name of the file in S3

``` python
def upload_files_to_s3(filename, bucket_name, key=None, ExtraArgs=None):
    """
    Uploads file to S3 bucket
    Args:
    filename: takes local filename to be uploaded
    bucker_name: name of the bucket into which the file is uploaded
    key: name of the file in the bucket. Default:None
    ExtraArgs: other arguments. Default:None
    """
    if key is None:
        key = filename
    
    try:
        s3client.upload_file(filename,bucket_name,key)
        print(f'uploaded file:{filename}')
    except botocore.exceptions.ClientError as e:
        print(e)
```

We can make use of `glob` module to upload multiple files in a folder

``` python
bucket1_files = [files[1],files[2]]
bucket2_files = [files[0],files[3]]
bucket1_files, bucket2_files
```

    (['data/30-days-create-folds.ipynb',
      'data/ARK_GENOMIC_REVOLUTION_ETF_ARKG_HOLDINGS.csv'],
     ['data/Player Data.xlsx', 'data/star_pattern_turtlesim.png'])

``` python
for file in bucket1_files:
    upload_files_to_s3(file,bucket_name=bucket_names[0])
```

    uploaded file:data/30-days-create-folds.ipynb
    uploaded file:data/ARK_GENOMIC_REVOLUTION_ETF_ARKG_HOLDINGS.csv

``` python
for file in bucket2_files:
    upload_files_to_s3(file,bucket_name=bucket_names[1])
```

    uploaded file:data/Player Data.xlsx
    uploaded file:data/star_pattern_turtlesim.png

## Get files list

Getting the files list from each bucket done using `list_objects` client function. It returns dict and we can iterate through `Contents` key to retrieve the filenames.

``` python
for bucket in bucket_names:
    print(f'Listing object inside bucket:{bucket}')
    list_obj_response = s3client.list_objects(Bucket=bucket)
    for obj in list_obj_response['Contents']:
        print(obj['Key'])
    print()
```

    Listing object inside bucket:my-s3bucket1-usohio-region
    data/30-days-create-folds.ipynb
    data/ARK_GENOMIC_REVOLUTION_ETF_ARKG_HOLDINGS.csv

    Listing object inside bucket:my-s3bucket2-usohio-region
    data/Player Data.xlsx
    data/star_pattern_turtlesim.png

## Download files

Downloading a file is very similar to uploading one. We need specify bucket name, name of the file to be downloaded, and the destination filename.

``` python
print(f"Downloading files from bucket:{bucket_names[1]}")
s3client.download_file(
    Bucket=bucket_names[1],
    Key="data/star_pattern_turtlesim.png",
    Filename="downloaded_turtlesim.jpg",
)
```

    Downloading files from bucket:my-s3bucket2-usohio-region

## Conclusion

This blog post shows how to use the boto3 python SDK to manage S3 aws service. With the help of documentation, we can implement require functionalities.

Back to top
