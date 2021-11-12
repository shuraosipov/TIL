# TIL
Today I Learned - a repository of things I learned today, with links and examples.

# [Lambda Best Practices] Separate the Lambda handler from your core logic (11 of November 2021)

While reading AWS Lambda best practices [0], I always ask myself why separate the Lambda handler from the core logic?
How does this affect the function performance, and why is it so important (so Amazon put this as a first item in the best practices section)?

Trying to answer that question I encountered different explanations like "This allows you to make a more unit-testable function", or "It is recommended to separate core logic from the Lambda handler, as the handler is generally used as an entry point to the function", or "This would also ease migrating your solution to any other platform if you need to". But still, I did not fully understand why.

To answer that question, I decided to understand Lambda's execution environment first. So, here is what I found.
According to the official documentation [1], the Lambda execution environment lifecycle [2] comprises three phases - Init, Invoke, and Shutdown. During the Init phase, Lambda creates or unfreezes an execution environment with the configured resources, downloads the code for the function and all layers, initializes any extensions, initializes the runtime, **and then runs the function's initialization code (the code outside the main handler).** The Init phase happens either during the first invocation or in advance of function invocations if you have enabled provisioned concurrency. 
Aha! So, translating this to plain English, it means that our code is pre-loaded and can be re-used by next function invocation.  

**During Invoke phase, Lambda invokes the function handler**. After the function runs to completion, Lambda prepares to handle another function invocation.

So, let's see how it works in practice. To do so, we will have two lambda functions, where the first function will have all code in the handler, and in the second, we will move function logic outside of the handler.

`Function A`
```
import json
import urllib.parse
import boto3

def lambda_handler(event, context):
    s3 = boto3.client('s3')
    
    def download_file(bucket, key, out_file):
        s3.download_file(bucket, key, out_file)
    
    def read_file(file):
        with open(file, 'r') as f:
            print(f.read())

    bucket = 'shuraosipov-content'
    key = 'sample_file.txt'
    out_file = '/tmp/sample_file.txt'
    
    download_file(bucket, key, out_file)
    read_file(out_file)
```

`Function B`
```
import json
import urllib.parse
import boto3

s3 = boto3.client('s3')

def download_file(bucket, key, out_file):
    s3.download_file(bucket, key, out_file)
    
def read_file(file):
    with open(file, 'r') as f:
        print(f.read())

def lambda_handler(event, context):
    
    bucket = 'shuraosipov-content'
    key = 'sample_file.txt'
    out_file = '/tmp/sample_file.txt'
    
    download_file(bucket, key, out_file)
    read_file(out_file)
```

Running both functions five times, I got the following results - in the first case, an initialization took 1954 ms, and further executions was around 350 ms. And in the second case, an initialization took about 320 ms, and further executions were completed in 120 ms.

`Function A`
```
Duration: 1954.15 ms	Billed Duration: 1955 ms	Memory Size: 128 MB	Max Memory Used: 73 MB	Init Duration: 332.99 ms

Duration: 350.79 ms	Billed Duration: 351 ms	Memory Size: 128 MB	Max Memory Used: 74 MB

Duration: 362.99 ms	Billed Duration: 363 ms	Memory Size: 128 MB	Max Memory Used: 74 MB

Duration: 356.27 ms	Billed Duration: 357 ms	Memory Size: 128 MB	Max Memory Used: 75 MB

Duration: 369.31 ms	Billed Duration: 370 ms	Memory Size: 128 MB	Max Memory Used: 75 MB
```

`Function B`
```

Duration: 321.61 ms	Billed Duration: 322 ms	Memory Size: 128 MB	Max Memory Used: 73 MB	Init Duration: 424.98 ms

Duration: 120.31 ms	Billed Duration: 121 ms	Memory Size: 128 MB	Max Memory Used: 73 MB

Duration: 129.14 ms	Billed Duration: 130 ms	Memory Size: 128 MB	Max Memory Used: 74 MB

Duration: 115.57 ms	Billed Duration: 116 ms	Memory Size: 128 MB	Max Memory Used: 74 MB

Duration: 110.96 ms	Billed Duration: 111 ms	Memory Size: 128 MB	Max Memory Used: 75 MB
```

## Links
[0] - https://docs.aws.amazon.com/lambda/latest/dg/best-practices.html#function-code\
[1] - https://docs.aws.amazon.com/lambda/latest/dg/runtimes-context.html\
[2] - Lambda invokes your function in an execution environment, which provides a secure and isolated runtime environment. The execution environment manages the resources required to run your function. The execution environment also provides lifecycle support for the function's runtime and any external extensions associated with your function.

# [ Databases ] How to validate data after a migration (10 of November 2021)
A customer is looking for a way to validate the data after the migration. They are replicating their production databases. Before doing the cutover, they want to have high confidence in their migration by comparing the data between the source and target.

The customer uses continuous replication to replicate data into different databases (for example, Oracle to PostgreSQL). Because it is their production system, they want to ensure that the data is being migrated without any loss or corruption.

## What I learned
To validate the replicated data, AWS DMS **compares the data between the source and target to ensure that the data is the same**. The tool **makes appropriate queries to the source and target to retrieve the data**. If the data volume is large, AWS DMS also adopts a strategy to **split the data into smaller manageable units and compare them**. AWS DMS splits the table into multiple smaller groups of contiguous rows based on the primary key. The tool then **compares the data at the partition level** and exposes results of the final comparison. This partitioning approach helps AWS DMS compare a predefined amount of data at any point in time.

## Links
- Blog: Migration Validation (Part 1) – Introducing Migration Assessment in AWS Database Migration Service: https://aws.amazon.com/blogs/database/migration-validation-part-1-introducing-migration-assessment-in-aws-database-migration-service/

- Blog: Migration Validation (Part 2) – Introducing Data Validation in AWS Database Migration Service: https://aws.amazon.com/blogs/database/migration-validation-part-2-introducing-data-validation-in-aws-data-migration-service/

- Blog: Validating Database Objects After Migration Using AWS SCT and AWS DMS: https://aws.amazon.com/blogs/database/validating-database-objects-after-migration-using-aws-sct-and-aws-dms/

# AWS Lambda Extensions (4 of November 2021)
An AWS Lambda extension is a companion process that augments the capabilities of a Lambda function. An extension can start before a function is invoked, run in parallel with a function, and continue to run after a function invocation is processed. In essence, a Lambda extension is like a client that runs in parallel to a Lambda invocation. This parallel client can interface with your function at any point during its lifecycle.

You can use Lambda extensions to augment your Lambda functions. For example, use Lambda extensions to integrate functions with your preferred monitoring, observability, security, and governance tools. You can choose from a broad set of tools that AWS Lambda Partners provides, or you can create your own Lambda extensions.

Lambda supports external and internal extensions. An external extension runs as an independent process in the execution environment and continues to run after the function invocation is fully processed. Because extensions run as separate processes, you can write them in a different language than the function.

As an example - "HashiCorp Vault allows you to secure, store, and tightly control access to your application’s secrets and sensitive data. With the Vault extension, you can now authenticate and securely retrieve dynamic secrets before your Lambda function invokes."

## What I learned
Extentions accessed by Lambda function similar to Lambda Layers.

AWS AppConfig has an available extension for .zip archive functions. This gives Lambda functions access to external configuration settings quickly and easily. **The extension runs a separate local process to retrieve and cache configuration data from the AWS AppConfig service. The function code can then fetch configuration data faster using a local call rather than over the network.**

You can deploy application configuration to multiple lambda functions by updating the extention (AppConfig for example), so you do not need to redeploy Lambda to apply a new settings.

Extensions share the same permissions as Lambda functions.

Extention can make calls to external/internal services and store response in a local cache so that Lambda will work with a local copy rather than making a call to remote services themself.


## Links
- https://aws.amazon.com/blogs/compute/introducing-aws-lambda-extensions-in-preview/
- https://docs.aws.amazon.com/lambda/latest/dg/using-extensions.html#using-extensions-reg
- https://github.com/aws-samples/aws-lambda-extensions
- https://aws.amazon.com/blogs/mt/introducing-aws-appconfig-lambda-extension-deploying-application-configuration-s
erverless/
- https://aws.amazon.com/systems-manager/features/appconfig
- https://docs.aws.amazon.com/appconfig/latest/userguide/appconfig-integration-lambda-extensions.html
- https://aws.amazon.com/blogs/aws/getting-started-with-using-your-favorite-operational-tools-on-aws-lambda-extensions-are-now-generally-available
- https://www.youtube.com/playlist?list=PLJo-rJlep0ECO8od7NRdfJ4OrnQ7TMAwj


# AWS Prescriptive Guidance (3 of November 2021)
Amazon Web Services (AWS) Prescriptive Guidance provides time-tested strategies, guides, and patterns to help accelerate your cloud migration, modernization, and optimization projects. These resources were developed by AWS technology experts and the global community of AWS Partners, based on their years of experience helping customers realize their business objectives on AWS.

https://aws.amazon.com/prescriptive-guidance


## What I learned
I was pleasantly surprised today when I found this resource - https://aws.amazon.com/prescriptive-guidance
I use https://aws.amazon.com/architecture/ and https://aws.amazon.com/solutions/ a lot, but this one was something new for me :clown_face:


# Instance Metadata Service Version 2 (IMDSv2) protocol for the EC2 instances (2 of November 2021)
IMDSv2 is session-based and requires you to authenticate requests to the instance metadata by using token. The token is a logical representation of a session. You will need to include the token in all GET requests to the instance metadata service.
You can generate a session token via PUT reqest to the token API. Token is instance-bound, i.e. it will work only from instance you generate it. Token TTL is vary from 1 sec to 6 hours. When token expires you must get another one (create a new session).

You can generate a token using the following command.
```
TOKEN=`curl -X PUT "http://169.254.169.254/latest/api/token" -H "X-aws-ec2-metadata-token-ttl-seconds: 21600"`
```

Then, use the token to generate top-level metadata items using the following command.
```
curl -H "X-aws-ec2-metadata-token: $TOKEN" -v http://169.254.169.254/latest/meta-data/
```

## What I learned
The metadata service was designed to be accessible only from within the instance. However, there is no additional access control to prevent unauthorized access from a compromised application running on the EC2 instance itself.

IMDSv2 protecting against misconfigured-open website application firewalls, misconfigured-open reverse proxies, unpatched SSRF vulnerabilities, and misconfigured-open layer-3 firewalls and network address translation.



## Links
- https://aws.amazon.com/blogs/security/defense-in-depth-open-firewalls-reverse-proxies-ssrf-vulnerabilities-ec2-instance-metadata-service/
- https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/configuring-instance-metadata-service.html
- https://www.youtube.com/watch?v=2B5bhZzayjI&ab_channel=AWSEvents


# Device Code Flow (1 of November 2021)
Device code flow also known as device authorization grant flow, or device flow.

On Microsoft identity platform you can use the OAuth 2.0 device authorization grant flow which allows users to sign in to input-constrained devices such as a browserless app, smart TV, IoT device, or printer. 

To enable this flow, the device has the user visit a webpage in their browser on another device to sign in. Once the user signs in, the device is able to get access tokens and refresh tokens as needed.

```
+----------+                                +----------------+
      |          |>---(A)-- Client Identifier --->|                |
      |          |                                |                |
      |          |<---(B)-- Device Code,      ---<|                |
      |          |          User Code,            |                |
      |  Device  |          & Verification URI    |                |
      |  Client  |                                |                |
      |          |  [polling]                     |                |
      |          |>---(E)-- Device Code       --->|                |
      |          |          & Client Identifier   |                |
      |          |                                |  Authorization |
      |          |<---(F)-- Access Token      ---<|     Server     |
      +----------+   (& Optional Refresh Token)   |                |
            v                                     |                |
            :                                     |                |
           (C) User Code & Verification URI       |                |
            :                                     |                |
            v                                     |                |
      +----------+                                |                |
      | End User |                                |                |
      |    at    |<---(D)-- End user reviews  --->|                |
      |  Browser |          authorization request |                |
      +----------+                                +----------------+

                    Figure 1: Device Authorization Flow
```

The device authorization flow illustrated in Figure 1 includes the following steps:
```
(A)  The client requests access from the authorization server and
        includes its client identifier in the request.

(B)  The authorization server issues a device code and an end-user
    code and provides the end-user verification URI.

(C)  The client instructs the end user to use a user agent on another
    device and visit the provided end-user verification URI.  The
    client provides the user with the end-user code to enter in
    order to review the authorization request.

(D)  The authorization server authenticates the end user (via the
    user agent), and prompts the user to input the user code
    provided by the device client.  The authorization server
    validates the user code provided by the user, and prompts the
    user to accept or decline the request.

(E)  While the end user reviews the client's request (step D), the
    client repeatedly polls the authorization server to find out if
    the user completed the user authorization step.  The client
    includes the device code and its client identifier.

(F)  The authorization server validates the device code provided by
    the client and responds with the access token if the client is
    granted access, an error if they are denied access, or an
    indication that the client should continue to poll.
```


## What I learned
You can use the Microsoft Authentication Library (MSAL) to acquire tokens from the Microsoft identity platform to authenticate users and access secured web APIs.

Whenever a user authentication is required, the command-line app provides a code and asks the user to use another device (such as an internet-connected smartphone) to navigate to https://microsoft.com/devicelogin, where the user will be prompted to enter the code. That done, the web page will lead the user through a normal authentication experience, including consent prompts and multi-factor authentication if necessary.

Upon successful authentication, the command-line app will receive the required tokens through a back channel and will use it to perform the web API calls it needs.


## Links
- https://github.com/Azure-Samples/ms-identity-python-devicecodeflow
- https://datatracker.ietf.org/doc/html/rfc8628
- https://docs.microsoft.com/en-us/azure/active-directory/develop/quickstart-v2-python-daemon


# Oracle Undo table (1 of November 2021)
Oracle Database creates and manages information used to roll back or undo changes to the database. Such information consists of records of the actions of transactions, primarily before they are committed. These records are collectively referred to as undo.

Undo records are used to:

* Roll back transactions when a ROLLBACK statement is issued
* Recover the database
* Provide read consistency
* Analyze data as of an earlier point in time by using Oracle Flashback Query
* Recover from logical corruptions using Oracle Flashback features

When a ROLLBACK statement is issued, undo records are used to undo changes that were made to the database by the uncommitted transaction. During database recovery, undo records are used to undo any uncommitted changes applied from the redo log to the datafiles. Undo records provide read consistency by maintaining the before image of the data for users who are accessing the data at the same time that another user is changing it.



## What I learned
We have an Oracle database for storing user actions (~ 100GB). Our application is configured to write user action logs to the database with the **retention policy of five years**. We use that data to analyze platform usage and auditing. 

To keep our environments as close to production as possible, we regularly copy that database to the downstream environments - dev, test, etc.
With one difference - application on downstream environments configured to **keep action logs for a six month**.

Today, we discovered that downstream environments stopped responding. After some investigation, we found that Undo tablespace ran out of space. It turned out that by changing the retention policy for action logs, we instructed Oracle to delete all data older than six months. And, according to the Oracle logic, it started saving that data to the Undo table for recovery, just in case we need to roll back the delete transactions later. 

Undo tablespace has retention period as well, which by default is 48 hours.

## Links
- https://docs.oracle.com/cd/E11882_01/server.112/e25494/undo.htm#ADMIN11460



