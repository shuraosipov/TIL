# TIL
Today I Learned - a repository of things I learned today, with links and examples.

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
https://docs.oracle.com/cd/E11882_01/server.112/e25494/undo.htm#ADMIN11460



