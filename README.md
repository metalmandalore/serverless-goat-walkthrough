#serverless-goat 
## The Verbose Walkthrough

*Still to add INSTALL/REMOVAL steps*

### Lambda Function API
1. Locate the Endpoint URL under CloudFormation Outputs
2. Navigate to URL in a separate browser  
The endpoint loads an OWASP ServerlessGoat page containing a form consisting of  
  *a single user input textbox and submit button  
  *a list of potential vulnerabilities  
This form was created to convert the URL of a Word Doc into a document on a webpage  
The textbox already contains the URL of a doc file  
3. Click Submit with the currrent value in place  
This loads a A Poison Tree poem with a URL that seems remarkably close to an s3 bucket address


