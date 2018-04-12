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

2. Click on **Users** and then **Add user**. Enter a name in the first field to remind you this User is the Framework, like `serverless-admin`. Enable **Programmatic access** by clicking the checkbox. 

3. Use narrowed permissions defined in [serverless-policy.json](serverless-policy.json) for this AWS user. Replace `${service}` with `clamav` - service name defined in [serverless.yml](src/serverless.yml). Thanks [unknown github user](https://github.com/serverless/serverless/issues/1439) for the help building it.

4. View and copy the **API Key** & **Secret** to a temporary place. 
    
5. Run 

    `serverless config credentials --provider aws --key <API key> --secret <Secret>`
    

That's it! Now your `serverless` cli is configured to work with your AWS account. You can change API key and Secret by running the same command with `-o` key (override).



### Build from Source

To build the functions and upload to AWS Lambda, run `make`.  The build process is completed using
the [amazonlinux](https://hub.docker.com/_/amazonlinux/) [Docker](https://www.docker.com)
 image.  The resulting files will be placed at `build/` during the `collect` stage. `serverless` will deploy it on AWS during the `deploy` stage.

### AV Defintion Bucket

Create an s3 bucket called `clamav.definition` to store current antivirus definitions.
This provides the fastest download speeds for the scanner.  This bucket can
be kept as private.  

To allow public access, useful for other accounts,
add the following policy to the bucket.

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "AllowPublic",
            "Effect": "Allow",
            "Principal": "*",
            "Action": [
                "s3:GetObject",
                "s3:GetObjectTagging"
            ],
            "Resource": "arn:aws:s3:::clamav.definition/*"
        }
    ]
}
```

### Definition Update Lambda

This function accesses the user’s ClamAV instance to download
updated definitions using `freshclam`.  It is recommended to run
this every 3 hours to stay protected from the latest threats.

The function is deployed and fully configured as `xxxx-update_av` Lambda. 
You can test it on aws or locally running the following command from the `build` directory:
```
serverless invoke -f update_av
```

It should return null. See logs on the CloudWatch for details.


### AV Scanner Lambda 

This function uploads all just downloaded files from configured buckets (see *S3 Events* below)
and sets 2 tags: `av-status` and `av-timestamp`. 
`av-status` is either `CLEAN` or `INFECTED`
`av-timestamp` is a scan time in format `2018/04/04 08:04:24 UTC`

### S3 Events

Configure scanning of buckets by adding a new S3 event to
invoke the Lambda function.  This is done from the properties of any
bucket in the AWS console.

![](../master/images/s3-event.png)

Note: If configured to update object metadata, events must only be
configured for `PUT` and `POST`. Metadata is immutable, which requires
the function to *copy* the object over itself with updated metadata. This
can cause a continuous loop of scanning if improperly configured.


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


## S3 Bucket Policy Examples

### Deny to download and re-tag "INFECTED" object
```
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Deny",
      "Action": ["s3:GetObject", "s3:PutObjectTagging"],
      "Principal": "*",
      "Resource": ["arn:aws:s3:::<<bucket-name>>/*"],
      "Condition": {
        "StringEquals": {
          "s3:ExistingObjectTag/av-status": "INFECTED"
        }
      }
    }
  ]
}
```

### (!!!DOESN'T WORK FOR ME!!!) Deny to download the object if not "CLEAN"
This policy doesn't allow to download the object until:
1) The lambda that run Clam-AV is finished (so the object has a tag)
2) The file is not CLEAN

Please make sure to check cloudtrail for the arn:aws:sts, just find the event open it and copy the sts.       
It should be in the format provided below:
```
 {
    "Effect": "Deny",
    "NotPrincipal": {
        "AWS": [
            "arn:aws:iam::<<aws-account-number>>:role/<<bucket-antivirus-role>>",
            "arn:aws:sts::<<aws-account-number>>:assumed-role/<<bucket-antivirus-role>>/<<bucket-antivirus-role>>",
            "arn:aws:iam::<<aws-account-number>>:root"
        ]
    },
    "Action": "s3:GetObject",
    "Resource": "arn:aws:s3:::<<bucket-name>>/*",
    "Condition": {
        "StringNotEquals": {
            "s3:ExistingObjectTag/av-status": "CLEAN"
        }
    }
}
```   



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
