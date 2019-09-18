#serverless-goat 
## The Verbose Walkthrough

*Still to add **INSTALL/REMOVAL** steps*

### Lambda Function API
1. Locate the Endpoint URL under CloudFormation Outputs
2. Navigate to URL in a separate browser  
The endpoint loads an OWASP ServerlessGoat page containing a form consisting of
- a single user input textbox and submit button  
- a list of potential vulnerabilities  
This form was created to convert the URL of a Word Doc into a document on a webpage  
The textbox already contains the URL of a doc file  
3. Click Submit with the currrent value in place  
This loads a A Poison Tree poem with a URL that seems remarkably close to an s3 bucket address

### Information Gathering
1. Click back on the browser to return to the page 
2. Attempt **Function Data Injection** by adding code to the end of the URL
e.g. try adding `; pwd`
3. Click submit to load the original poem as well as garbeled information 
  *this appears to be vulnerable to __Function Data Injection__*
#### Improved injection attempts
1. Click back and change the URL form to a URL that doesn't contain a doc file followed by a snippet of code . 
e.g. `https://inject; pwd`
2. Click submit to load the current working directory */var/task*
3. Repeat this tasks with the following bash code add for more information  
**Document everything for later**    
`https://inject; whoami`
   *current user should be sbx-user0666 or similar*
`https://inject; ls -la`
    *lists the current directory, which shows index.js, as well as a node_modules subdirectory*
`https://inject; cat /etc/passwd`
  *lists current users*  
`https://inject; cat index.js`
    *Provides code for basic information on how this page functions*   
    Important information to be gathered from this file includes
      * Database 
      * S3 bucket Put permissions
      * node.js package used 
      * Table information 
      * Permissions to curl (potentially) 
      * Output URL 
 
