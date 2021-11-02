# TIL
Today I Learned - a repository of things I learned today, with links and examples.



# Device Code Flow (1 of November 2021)
On Microsoft identity platform you can use the OAuth 2.0 device authorization grant flow which allows users to sign in to input-constrained devices such as a browserless app, smart TV, IoT device, or printer. 

To enable this flow, the device has the user visit a webpage in their browser on another device to sign in. Once the user signs in, the device is able to get access tokens and refresh tokens as needed.

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



