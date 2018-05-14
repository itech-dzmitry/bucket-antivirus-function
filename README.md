# bucket-antivirus-function

[![CircleCI](https://circleci.com/gh/upsidetravel/bucket-antivirus-function.svg?style=svg)](https://circleci.com/gh/upsidetravel/bucket-antivirus-function)

Scan new objects added to any s3 bucket using AWS Lambda. [more details in this post](https://engineering.upside.com/s3-antivirus-scanning-with-lambda-and-clamav-7d33f9c5092e)

## Features

- Easy to install
- Send events from an unlimited number of S3 buckets
- Prevent reading of infected files using S3 bucket policies
- Accesses the end-user’s separate installation of
open source antivirus engine [ClamAV](http://www.clamav.net/)

## Requirements
- make
- [docker](https://www.docker.com)
- [serverless](https://serverless.com)


## How Does It Work?

![](../master/images/bucket-antivirus-function.png)

- Each time a new object is added to a bucket, S3 invokes the Lambda
function to scan the object
- The function package will download (if needed) current antivirus
definitions from a S3 bucket. Transfer speeds between a S3 bucket and
Lambda are typically faster and more reliable than another source
- The object is scanned for viruses and malware.  Archive files are
extracted and the files inside scanned also
- The objects tags are updated to reflect the result of the scan, CLEAN
or INFECTED, along with the date and time of the scan.
- Object metadata is updated to reflect the result of the scan (optional)
- Metrics are sent to [DataDog](https://www.datadoghq.com/) (optional)
- Scan results are published to a SNS topic (optional)

## Installation

### Configure AWS+serverless
1. Login to your Amazon Web Services Account and go to the Identity & Access Management (IAM) page.

2. Click on **Users** and then **Add user**. 
Enter a name in the first field to remind you this User is the Framework, like `serverless-admin`. 
Enable **Programmatic access** by clicking the checkbox. 

3. Use narrowed permissions defined in [serverless-policy.json](serverless-policy.json) for this AWS user. 
Thanks [unknown github user](https://github.com/serverless/serverless/issues/1439) for the help building it.

4. View and copy the **API Key** & **Secret** to a temporary place. 
    
5. Run 

    `serverless config credentials --provider aws --key <API key> --secret <Secret>`
    

That's it! Now your `serverless` cli is configured to work with your AWS account. 
You can change API key and Secret by running the same command with `-o` key (override).


### AV Defintion Bucket

Choose any unique name for the bucket to store current antivirus definitions. This bucket can
be kept as private.

**Set environment variable** (!!!!):

`AV_DEFINITION_S3_BUCKET=JUST_CREATED_BUCKET_NAME`


### Build from Source

To build the functions and upload to AWS Lambda, run `make`.  The build process is completed using
the [amazonlinux](https://hub.docker.com/_/amazonlinux/) [Docker](https://www.docker.com) image.  
The resulting files will be placed at `build/` during the `collect` stage. 
`serverless` will deploy it on AWS during the `deploy` stage.

After the deployment 2 lambda functions will be added to your AWS account: **update_av** and **scan** 


### Definition Update Lambda (update_av)

This function accesses the user’s ClamAV instance to download
updated definitions using `freshclam`.  

The function is deployed and configured to run once per 3 hours as `clamav-xxxx-update_av` Lambda.
 
Run it in either of 2 ways to be sure it deployed successfully:
 * AWS Lambda *Test* button 
 * Run the following command from the local `build` directory:
 
     `serverless invoke -f update_av`

It should return null. See logs on the **CloudWatch** for details in case of any issue.


### Subscribe to the buckets updates. 

Configuration of buckets to be scanned consists of 2 steps:

* Configuring the `scan` lambda policy.
* Configuring s3 event.


**The scan lambda policy**

* Open AWS IAM service.
* Choose the `Roles` menu.
* Select `clamav-dev-scan` role and edit its policy.
* Find a policy with `"Sid": "AccessBucketsToScan"`. 
* Find a default bucket in the `Resource` part: `"arn:aws:s3:::bucket-to-scan/*"`. 
* Replace `bucket-to-scan` with the target bucket. 
* Add new buckets into the list in the same way. 


**S3 event**

Add a new S3 event to invoke the **scan** Lambda function.  This is done from the properties of any
bucket in the AWS console.

![](../master/images/s3-event.png)



### Run it!

Upload a couple of files into the bucket you've just configured. 
Use [EICAR test file](https://en.wikipedia.org/wiki/EICAR_test_file) as a sample infected file.

S3 event invokes the **scan** lambda which sets following 2 tags:
  
| Tag | Value |
| --- | --- |
| `av-status` | either `CLEAN` or `INFECTED` |
| `av-timestamp` | a scan time in format `2018/04/04 08:04:24 UTC` |


See logs on the **CloudWatch** for details.


### How to remove it?

Run `serverless remove` from the local `build` directory to remove the antivirus.



## Configuration

Runtime configuration is accomplished using environment variables.  See
the table below for reference.

| Variable | Description | Default | Required |
| --- | --- | --- | --- |
| AV_DEFINITION_S3_BUCKET | Bucket containing antivirus definition files |  | Yes |
| AV_DEFINITION_S3_PREFIX | Prefix for antivirus definition files | clamav_defs | No |
| AV_DEFINITION_PATH | Path containing files at runtime | /tmp/clamav_defs | No |
| AV_SCAN_START_SNS_ARN | SNS topic ARN to publish notification about start of scan | | No |
| AV_SCAN_START_METADATA | The tag/metada indicating the start of the scan | av-scan-start | No |
| AV_STATUS_CLEAN | The value assigned to clean items inside of tags/metadata | CLEAN | No |
| AV_STATUS_INFECTED | The value assigned to clean items inside of tags/metadata | INFECTED | No |
| AV_STATUS_METADATA | The tag/metadata name representing file's AV status | av-status | No |
| AV_STATUS_SNS_ARN | SNS topic ARN to publish scan results (optional) | | No |
| AV_TIMESTAMP_METADATA | The tag/metadata name representing file's scan time | av-timestamp | No |
| CLAMAVLIB_PATH | Path to ClamAV library files | ./bin | No |
| CLAMSCAN_PATH | Path to ClamAV clamscan binary | ./bin/clamscan | No |
| FRESHCLAM_PATH | Path to ClamAV freshclam binary | ./bin/freshclam | No |
| DATADOG_API_KEY | API Key for pushing metrics to DataDog (optional) | | No |
| AV_PROCESS_ORIGINAL_VERSION_ONLY | Controls that only original version of an S3 key is processed (if bucket versioning is enabled) | False | No |




## License

```
Upside Travel, Inc.
Modified by dzmitry.dziamidau@itechart-group.com

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.
```

ClamAV is released under the [GPL Version 2 License](https://github.com/vrtadmin/clamav-devel/blob/master/COPYING)
and all [source for ClamAV](https://github.com/vrtadmin/clamav-devel) is available
for download on Github.
