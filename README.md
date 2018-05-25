# Introduction: **Build a Modern Application on AWS (Python)**

**AWS Experience: Beginner**
**Time to Complete: 2 hours**
**$ Cost to Complete: Many of the services used are included in the AWS Free Tier. For those that are not, the sample application will cost, in total, less than $1/day. ***
**Tutorial Prereqs: **

* **An AWS Account**

* [TODO] **Standard Disclaimers on typical usage and needing to terminate.**

### **Overview**

AWS provides all the tools and services required for a developer to build a modern application.  This tutorial will walk you through the steps to create a sample web application that leverages modern components and approaches such as container-deployed microservices, infrastructure as code, CI/CD, and serverless.  You will build, from the ground up, a sample website called Mythical Mysfits designed to enable visitors to adopt a fantasy creature as a pet.

The site will present mysfits available for adoption with some different characteristics about each. Users will be able to vote on which mysfits are their favorites, and then upon login users can choose to adopt the mysfit they'd like to reserve for adoption.  The Mythical Mysfits website you create will also allow you to gather insights about user behavior for future analyses.

### Application Architecture

The website serves it's static content directly from Amazon S3, provides a microservice API backend deployed as a container through AWS Fargate on Amazon ECS, stores data in a managed NoSQL database provided by Amazon DynamoDB, with authentication and authorization for the application enabled through AWS API Gateway and it's integration with Amazon Cognito.  The user behavior will be sent as records to an Amazon Kinesis stream where those records will be processed by serverless AWS Lambda functions and eventually stored in Amazon S3 via an Amazon Kinesis Firehose.

You will be creating and deploying changes to this application completely programmatically.  We have provided AWS CloudFormation templates that define the required infrastructure components as code, which includes a fully managed CI/CD stack utilizing AWS CodeCommit, CodeBuild, and CodePipeline.  Finally, you will complete the development tasks required all within your own browser by leveraging the cloud-based IDE, AWS Cloud9.



# Module 1: IDE Setup and Static Website Hosting

Time to complete: 20 minutes
Services used: AWS Cloud9, Amazon S3

In this module, follow the instructions to create your cloud-based IDE on AWS Cloud9 and deploy the first version of the static Mythical Mysfits website.

### Select a Region

This web application can be deployed in any AWS region that supports all the services used in this application, which include Amazon Cognito, AWS Lambda, and Amazon Lex. Refer to the region table to see which regions have the supported services. The supported regions include:

* US East (N. Virginia)
* EU (Ireland)

Select a region from the dropdown in the upper right corner of the AWS Management Console.

### Create a new AWS Cloud9 Environment

 On the AWS Console home page, type “Cloud9” into the service search bar and select it:

Click “Create Environment” on the Cloud9 welcome page:

Name your environment “MythicalMysfitsIDE” with any description you'd like, and click “Next Step””
Leave the Environment settings as their defaults and click “Next Step”:
Click “Create Environment”:

When the IDE has finished being created for you, you'll be presented with a welcome screen that looks like this:
In the bottom panel, you will see a terminal command line open and read to use.  Run the following git command in the terminal to clone the necessary code to complete this tutorial:

```
git clone https://github.com/aws-samples/aws-modern-application-workshop.git
```

After cloning the repository, you'll see that your project explorer now includes the files cloned:


In the terminal, change directory to the newly cloned repository directory:

```
cd aws-modern-application-workshop
```

Next, we will create the infrastructure components needed for hosting a static website in Amazon S3 via AWS CloudFormation.  In the cloned repository, we have included a CloudFormation template that can create the Amazon S3 bucket required, as well as configure it to be usable for static website hosting.  To create the stack represented by the CloudFormation template, run the following CloudFormation command via the AWS Command Line Interface:

```
aws cloudformation create-stack --stack-name MythicalMysfitsWebsiteBucket --template-body file://cfn-templates/s3-website.yml
```

The output of this command will indicate a new stack is being created by CloudFormation with the StackId in the response:

```
{
    "StackId": "arn:aws:cloudformation:us-east-1:xxxx:stack/MythicalMysfitsWebsiteBucket/xxxx"
}
```

You can either visit the CloudFormation console to view the creation status of your stack, or execute the following CLI command:

```
aws cloudformation describe-stacks --stack-name MythicalMysfitsWebsiteBucket
```

When you see the StackStatus of “CREATE_COMPLETE”, your S3 bucket has been created.  Find the “Outputs” object within the response to find the HTTPS URL that you can use to access your new website, save this for reference.  Also, within the URL you can find the name of the S3 bucket that has been created.  Save this for reference as well, the bolded section below represents the bucket name that can be found within the Output S3BucketSecureURL:

```
{
                    "Description": "Name of S3 bucket to hold website content",
                    "OutputKey": "S3BucketSecureURL",
                    "OutputValue": "https://_**mythicalmysfitswebsitebucket-s3bucket-xxxxx**_.amazonaws.com"
                }
```


Now we need to copy the first version of Mythical Mysfits homepage to the bucket.  This is included as an index.html file within the /module-1/web/ directory of the repository you cloned.  We will accomplish this using the AWS CLI using the following command, which will use the S3 bucket name that you save from above (the bolded component):

```
aws s3 cp ./module-1/web/index.html s3://_**mythicalmysfitswebsitebucket-s3bucket-xxxx**_/index.html
```

Now, if you visit the full URL saved earlier, you can see that the initial Mythical Mysfits website is up and running!

[TODO insert image of website]

That concludes Module 1.


# Module 2: Creating a Service with AWS Fargate

In Module 2, you will create a new microservice hosted with AWS Fargate on Amazon Elastic Container Service so that your Mythical Mysfits website can have a application backend to integrate with. AWS Fargate is a feature of Amazon ECS that allows you to deploy containers without having to manage any clusters or servers. For our Mythical Mysfits backend, we will use Python and create a Flask app in a Docker container behind a Network Load Balancer. These will form the microservice backend for the frontend website to integrate with.

### Module 2a: Creating the Core Infrastructure

Before we can create our service, we need to create the core infrastructure environment that the service will use, including the networking infrastructure in Amazon VPC, and the AWS Identity and Access Management Roles that will define the permissions that ECS and our containers will have on top of AWS.  It is common on many teams to have separate teams with elevated access in AWS that are responsible for creating and modifying Network and Security resources. We have followed that model here to demonstrate how CloudFormation can help enforce separation of duties on AWS for your team through modular templates.  We have provided a CloudFormation template to create all of the necessary Network and Security resources in /cfn-templates/core.yml.  This template will create the following resources:

* An Amazon VPC - a network environment that contains four subnets (two public and two private) in the 10.0.0.0/16 private IP space, as well as all the needed Route Table configurations.
* Two NAT Gateways (one for each public subnet) - allows the containers we will eventually deploy into our private subnets to communicate out to the Internet to download necessary packages, etc.
* A DynamoDB Endpoint - our microservice backend will eventually integrate with Amazon DynamoDB for persistence (as part of module 3).
* A Network Load Balancer - A high throughput and low latency TCP load balancer that will route requests from the Internet to your service containers.
* A Security Group - Allows your docker containers to receive traffic on port 8080 from the Internet through the Network Load Balancer.
* Two IAM Roles - Two Identity and Access Management Roles are created. One for the Amazon ECS service that allow it to interact with the infrastructure environment as needed. Another that will provide your docker containers with the permissions they require for interacting with AWS services like DynamoDB and CloudWatch Logs.

To create these resources, run the following command in the Cloud9 terminal (will take ~10 minutes for stack to be created):

```
aws cloudformation create-stack --stack-name MythicalMysfitsCoreStack --capabilities CAPABILITY_IAM --template-body file://cfn-templates/core.yml   
```

Remember you can check on the status of your stack creation either via the AWS Console or by running the command:

```
aws cloudformation describe-stacks --stack-name MythicalMysfitsCoreStack
```

### module 2B: Creating your First Docker Image

Next, you will create a docker container image that contains all of the code and configuration required to run the Mythical Mysfits backend as API created with Flask.  We will build the container image within Cloud9 and then push it to the Amazon Elastic Container Registry, where it will be available to pull when we create our service using Fargate.

All of the code required to run our service backend is stored within the /module-2/app/ directory of the repository you've cloned into your Cloud9 IDE.  If you would like to review the Python code that uses Flask to create the service API, view the /module-2/app/service/mythicalMysfitsService.py file.

Docker comes already installed on the Cloud9 IDE that you've created, so in order to build the docker image locally, all we need to do is run the following commands in the Cloud9 terminal:

* First change directory to ~/environment/module-2/app

```
cd ~/environment/aws-modern-application-workshop/module-2/app
```

* Then build the docker image, this will use the file in the current directory called Dockerfile that tells Docker all of the instructions that should take place when the build command is executed. Replace the contents in {braces} below with the appropriate information from the account/region you're working in:

```
docker build . -t {aws_account_id}.dkr.ecr.{us-east-1}.amazonaws.com/mythicalmysfits/service:latest
```

You will see docker download and install all of the necessary dependency packages that our application needs, and output the tag for the built image.  Copy the image tag for later reference.

```
Successfully built 8bxxxxxxxxab
Successfully tagged _**111111111111.dkr.ecr.us-east-1.amazonaws.com/mythicalmysfits/service:latest**_
```

Let's test our image locally within Cloud9 to make sure everything is operating as expected. Copy the image tag that resulted from the previous camm and run the following command to deploy the container “locally” (which is actually within your Cloud9 IDE inside AWS!):

```
docker run -p 8080:8080 _**111111111111.dkr.ecr.us-east-1.amazonaws.com/mythicalmysfits/service:latest**_
```

As a result you will see docker reporting that your container is up and running locally:

```
 * Running on http://0.0.0.0:8080/ (Press CTRL+C to quit)
```

To test our service with a local request, we're going to open up a second terminal window within Cloud9 where we can run additional commands while our service is running.  To open a new terminal in Cloud9 click the '+' symbol next to the existing terminal window and select “New Terminal”:

Within the new terminal window, we will use cURL to send a request to our locally running service:

```
curl 0.0.0.0:8080/mysfits
```

If successful you will see a response from the service that returns the JSON document stored at /modern-application-workshop/module-2/app/service/mysfits-response.json

With a successful test of our service locally, we're ready to create a container image repository in Amazon ECR and push our image into it.  In order to create the registry, run the following command, this creates a new repository in the default AWS ECR registry created for your account.

```
aws ecr create-repository --repository-name mythicalmysfits/service
```

The response to this command will contain additional metadata about the created repository.
In order to push images into our new repository, we will need to obtain authentication credentials for our Docker client to the repository.  Run the following command, which will return a login command to retrieve credentials for our Docker client and then automatically execute it (include the full command including the $ below). 'Login Succeeded' will be reported if the command is successful.

```
$(aws ecr get-login --no-include-email)
```

Next, push the image you created to the Amazon ECR repository using the copied tag from above. Using this command, docker will push your image and all the images it depends on to Amazon ECR:

```
docker push _**111111111111.dkr.ecr.us-east-1.amazonaws.com/mythicalmysfits/service:latest**_
```

Run the following command to see your newly pushed docker image stored inside the ECR repository:

```
aws ecr describe-images --repository-name mythicalmysfits/service
```

### Module 2C: Creating your first Fargate Service

Now,  we have an image available in ECR that we can deploy to a service hosted on Amazon ECS using AWS Fargate.  The same service you tested locally via the terminal in Cloud9 as part of the last module will now be deployed in the cloud and publicly available behind a Network Load Balancer.  We have provided a CloudFormation template to accomplish this, representing all of the infrastructure you need as code.  This CloudFormation template will create the following resources:

* An ECS Cluster - The cluster of “servers” that your service containers will be deployed to.  Servers is in quotations here because you will in fact be using AWS Fargate, which allows you to specify that your containers be deployed to a cluster without having to actually provision or manage any servers yourself.
* An AWS CloudWatch Logs Group - The logs that your container generates will automatically be pushed to AWS CloudWatch logs as part of this group. Especially important when using AWS Fargate since you will not have access to the server infrastructure where your containers are running.
* An ECS Task Definition - A Task in ECS defines one or more containers that are deployed together to a cluster and the resources and configuration options that those containers require to run.  The container included in this task definition is the image that we pushed into ECR as part of the last module.
* An ECS Service - The service itself.  Here we define that the Task above will be what is deployed and run as the service, that the Tasks should be deployed to Fargate so that they may be run serverless, indicate that this service and it's containers should be registered to the network load balancer and allow the traffic we permitted in the Security Groups created with the core.yml CloudFormation template used in Module 2A.

To create these resources using the CloudFormation template included, run the following command in the Cloud9 terminal:

```
aws cloudformation create-stack --stack-name MythicalMysfitsServiceStack --template-body file://~/environment/modern-application-workshop/cfn-templates/service.yml
```

Once the stack has been created, let's try sending a request to the network load balancer (NLB) to confirm our service is up and available.  To find the URL of the NLB, execute the following command in the terminal to see the outputs we have configured in the MythicalMysfitsCoreStack we created earlier with CloudFormation:

```
aws cloudformation describe-stacks --stack-name MythicalMysfitsCoreStack
```

In the response JSON, you will see an Output listed called “ExternalUrl”, whose OutputValue is the URL to be used to send a request to the created NLB.  See below:

```
{
                    "Description": "The url of the external load balancer",
                    "ExportName": "MythicalMysfitsCoreStack:ExternalUrl",
                    "OutputKey": "ExternalUrl",
                    "OutputValue": "_**http://Mythi-Publi-123456789-abc123456.elb.us-east-1.amazonaws.com**_"
                }
```

Let's copy that URL and send a request to it using the cURL command again in the terminal (or by simply a web browser, since this time our service is available on the Internet):

```
curl http://Mythi-Publi-123456789-abc123456.elb.us-east-1.amazonaws.com/mysfits
```

A response showing the same JSON response we received earlier when testing the docker container locally in Cloud9 means your Flask API is up and running on AWS Fargate.

Next, we need to integrate our website with your new API backend instead of using the hard coded data that we previously uploaded to S3.  You'll need to update the following file to use the same NLB URL for API calls (do not inlcude the /mysfits path): /module-2/web/index.html
Open the file in Cloud9 and replace the highlighted area below between the quotes with the NLB URL:


After pasting, the line should look similar to below:
To upload this file to your S3 hosted website, use the bucket name again that was created during Module 1, and run the following command:

```
aws s3 cp ~/environment/modern-application-workshop/module-2/web/index.html s3://INSERT-YOUR-BUCKET-NAME/index.html
```

 Open your website using the same URL used at the end of Module 1 in order to see your new Mythical Mysfits website, which is retrieving JSON data from your Flask API running within a docker container deployed to AWS Fargate!

### Module 2D: Automating Deployments using AWS Code Services

Now that you have a service up and running, you may think of code changes that you'd like to make to your Flask service.  It would be a bottleneck for your development speed if you had to go through all of the same steps above every time you wanted to deploy a new feature to your service. That's where Continuous Integration and Continuous Delivery or CI/CD come in!

In this module, you will create a fully managed CI/CD stack that will automatically deliver all of the code changes that you make to your code base to the service you created during the last module.  To help you accomplish this, we have included another CloudFormation template that will create the following resources for you:

* An AWS CodeCommit Repository: A managed and private Git repository where your code will be stored.
* An AWS CodeBuild Project: Will automatically provision a build server to our configuration and execute the steps required to build our docker image and push a new version of it to the ECR repository we created.  These steps are included in the /module-3/app/buildspec.yml file.  The buildspec.yml file is what you create to instruct CodeBuild what steps are required for a build execution within the service.
* An Amazon ECR Repository - We will recreate the repository you created by through the command line during the last module so that all of the pieces of our CI/CD stack can be represented by the same CloudFormation template.
* An AWS CodePipeline Pipeline: Orchestrates the entire CI/CD process from beginning to end.  Detects when changes are pushed into our CodeCommit repository, triggers a build to occur for those changes in our CodeBuild project, and then takes those completed changes and deploys them to the ECS service we created above in Module 2.
* IAM Roles - IAM roles required for the services above to execute within your account with the required permissions.

First, let's delete the repository you created by hand during the last module so that CloudFormation can create it anew as part of this holistic CI/CD stack.  Run the following in the terminal:

```
aws ecr delete-repository —repository-name mythicalmysfits/service —force
```

**Note:** In a real-world scenario, you would more typically create this CI/CD stack as development begins and then create the service stack once you need to first deploy the code you've written.  But, for this workshop we had you create the service first to become familiar with its concepts before deploying to it with automation.

To use CloudFormation to create the CI/CD stack for your Mythical Mysfits service, run the following command in the terminal:

```
aws cloudformation create-stack --stack-name MythicalMysfitsCICDStack --capabilities CAPABILITY_NAMED_IAM --template-body file://~/environment/modern-application-workshop/cfn-templates/devtools.yml
```

When that stack has been created successfully, we first must integrate our Cloud9 IDE with the new CodeCommit code repository that was just created.

First, we need to generate git credentials to interact with the created repository. AWS CodeCommit provides a credential helper for git that we will use to make integration easy.  Run the following commands in sequence the terminal (neither will report any response if successful):

```
`git config --global user.name "*Your Name*"`
```

```
git config --global user.email your.name*`@example.*com*`*
```

```
git config --global credential.helper '!aws codecommit credential-helper $@'
```

```
git config --global credential.UseHttpPath true
```

Next change directories in your IDE to the environment directory using the terminal:

```
cd ~/environment/
```

Now, we are ready to clone our repository using the following terminal command (be sure to replace us-east-1 with the region you are using for the workshop, if needed):

```
git clone https://git-codecommit.us-east-1.amazonaws.com/v1/repos/MythicalMysfitsService-Repository
```

This will tell us that our repository is empty!  Let's fix that by copying the application files into our repository directory using the following command:

```
cp -r ~/environment/modern-application-workshop/module-2/app/* ~/environment/MythicalMysfitsService-Repository
```

Now the completed service code that we used to create our Fargate service in Module 2 is stored in the local repository that we just cloned from AWS CodeCommit.  Let's make a change to the Flask service before committing our changes, to demonstrate that the CI/CD pipeline we've created is working. In Cloud9, open the file stored at **~/environment/MythicalMysfitsService-Repository/service/mysfits-response.json** and change the age of one of the mysfits to another value and save the file.

After saving the file, change directories to the new repository directory:

```
cd ~/environment/MythicalMysfitsService-Repository/
```

Then, run the following git commands to push in your code changes.  

```
git add .
git commit -m "I changed the age of one of the mysfits."
git push
```

After they are pushed in to the repository, you can open the CodePipeline service in the AWS Console to view your changes as they progress through the CI/CD pipeline. After committing your code change, it will take about 5 to 10 minutes for the changes to be deployed to your live service running in Fargate. Refresh your Mythical Mysfits website in the browser to see that the changes have taken effect.

This concludes Module 2.

# Module 3 - Adding a Data Tier with Amazon DynamoDB

Now that you have a service deployed and a working CI/CD pipeline to deliver changes to that service automatically whenever you update your code repository, you can quickly move new application features from conception to available for your Mythical Mysfits customers.  With this increased agility, let's add another foundational piece of functionality to the Mythical Mysfits website architecture, a data tier.  In this module you will create a table in Amazon DynamoDB, a managed and scalable NoSQL database service on AWS with super fast performance.  Rather than have all of the Mysfits be stored in a static JSON file, we will store them in a database to make the websites future more extensible and scalable.

### Creating A DynamoDB Table using Cloudformation

To add the DynamoDB table to the architecture, we have included another CloudFormation template that contains the resource definition required to create a table called **MysfitsTable**. This table will have a primary index defined by a hash key attribute called **MysfitId,** and two more secondary indexes.  The first secondary index will have the hash key of **Species** and a range key of **MysfitId, **and the second secondary index will have the hash key of **Alignment** and a range key of **MysfitId.  **These two secondary indexes will allow us to execute queries against the table to retrieve all of the mysfits that match a given Species or Alignment to enable the filter functionality you may have noticed isn't yet working on the website.

To create the table using CloudFormation, execute the following command in the Cloud9 terminal:

```
aws cloudformation update-stack --stack-name MythicalMysfitsServiceStack --template-body file://~/environment/modern-application-workshop/cfn-templates/service-with-ddb.yml
```

After the stack completes updating, you can view your newly created table either in the DynamoDB  console or by executing the following AWS CLI command in the terminal:

```
aws dynamodb describe-table --table-name MysfitsTable
```

If we execute the following command to retrieve all of the items stored in the table, you'll see that the table is empty:

```
aws dynamodb scan --table-name MysfitsTable
```

```
{
    "Count": 0,
    "Items": [],
    "ScannedCount": 0,
    "ConsumedCapacity": null
}
```

Also provided is a JSON file that can be used to batch insert a number of Mysfit items into this table.  This will be accomplished through the DynamoDB API **BatchWriteItem. **To call this API using the provided JSON file, execute the following terminal command (the response from the service should report that there are no items that went unprocessed):

```
aws dynamodb batch-write-item --request-items file://~/environment/modern-application-workshop/lib/populate-dynamodb.json
```

Now, if you run the same command to scan all of the table contents, you'll find the items have been loaded into the table:

```
aws dynamodb scan --table-name MysfitsTable
```

### Committing your first Code change

Now that we have our data included in the table, let's modify our application code to read from this table instead of returning the static JSON file response that was used in Module 2.  We have included a new set of Python files for your Flask microservice, but now instead of reading the static JSON file will make a request to DynamoDB.

The request is formed using the AWS Python SDK called **boto3.** This SDK is a powerful yet simple way to interact with AWS services via Python code. It enables you to use service client definitions and functions that have great symmetry with the AWS APIs and CLI commands you've already been executing as part of this workshop.  Translating those commands to working Python code is simple when using boto3**.  **To copy the new files into your CodeCommit repository directory, execute the following command in the terminal:

```
cp ~/environment/modern-application-workshop/module-3/app/service/* ~/environment/MythicalMysfitsService-Repository/service/
```

Now, we need to check in these code changes to CodeCommit using the git command line client.  Run the following commands to check in the new code changes and kick of your CI/CD pipeline:

```
cd ~/environment/MythicalMysfitsRepository
```

```
git add .
```

```
git commit -m "Add new integration to DynamoDB."
```

```
git push
```

Now, in just 5-10 minutes you'll see your code changes make it through your full CI/CD pipeline in CodePipeline and out to your deployed Flask service to AWS Fargate on Amazon ECS.

### Update The Website Content in S3

Finally, we need to publish a new index.html page to our S3 bucket so that the new API functionality using query strings to filter responses will be used.  The new index.html file is located at ~/environment/modern-application-workshop/module-3/web/index.html.  Open this file in your Cloud9 IDE and replace the string indicating “REPLACE_ME” just as you did in Module 1, with the appropriate NLB endpoint.  Refer to the file you already edited in the /module-1/ directory if you need to.  After replacing the endpoint to point at your NLB, upload the new index.html file by running the following command (replacing with the name of the bucket you created in Module 1:

```
aws s3 cp ~/environment/modern-application-workshop/module-3/web/* s3://{your_bucket_here}/
```

Re-visit your Mythical Mysfits website to see the new population of Mysfits loading from your DynamoDB table and how the Filter functionality is working!

That concludes module 3.

# Module 4: Adding User and API features with Amazon API Gateway and AWS Cognito

In order to add some more critical aspects to the Mythical Mysfits website, like allowing users to vote for their favorite mysfit and adopt a mysfit, we need to first have users register on the website.  To enable registration and authentication of website users, we will create a User Pool in AWS Cognito - a fully managed user identity management service .  Then, to make sure that only registered users are authorized to like or adopt mysfits on the website, we will deploy an API with Amazon API Gateway to sit in front of our network load balancer. Amazon API Gateway is also a managed service, and provides critical REST API capabilities out of the box like SSL termination, request authorization, request throttling, and much more.
You will again use infrastructure as code through CloudFormation to deploy the needed resources to AWS. The CloudFormation template for this module is called **service-with-apigw.yml**. It contains the same resources as the previous Service CloudFormation templates with the following additions:

* The** Cognito User Pool **described above.
* The **Amazon API Gateway** **REST API** described above.
* An **Amazon API Gateway VPC Link** that enables APIs created with API Gateway to privately communicate with a service fronted by a Network Load Balancer like we deployed during this workshop.  **Note: **For the purposes of this workshop, we created the NLB to be internet-facing so that it could be called directly in earlier modules. So even though we will be requiring Authorization tokens in our API after this module, our NLB will still actually be open to the public behind the API Gateway API.  In a real-world scenario, you would have created your NLB to be internal from the beginning, knowing that API Gateway was your strategy for Internet-facing API authorization.

Another change to this CloudFormation template uses the AWS **Serverless Application Model (SAM)** to define the new API. You can see this by looking at the second line of the template, where a Transform section is defined:

```
Transform: AWS::Serverless-2016-10-31
```

This tells CloudFormation that our template should be transformed using the specified version of SAM.  SAM gives us the ability to simply define serverless resources like APIs and Lambda functions using simplified resource definitions in JSON or YAML.  SAM also provides additional capabilities related to the packaging and testing of Lambda functions, which you'll see later in Module 5.  For this module, we've just used SAM to let us define our API in-line using Swagger 2.0.  In order to push this update out to the service stack in CloudFormation, we'll use a different command than before, called **deploy.**

This command will take the SAM template that we have created and transform it into typical CloudFormation and generate the change set to be applied to our stack, then subsequently execute the changes to the stack using the same update command we used in the last module.  To add the Cogntio User Pool and API Gateway API to our service stack run the following command in your terminal:

```
aws cloudformation deploy --stack-name MythicalMysfitsServiceStack --template-file ~/environment/modern-application-workshop/cfn-templates/service-with-apigw.yml
```

Once the stack has updated, run the following command to show the Output of the CloudFormation stack.  This latest version includes an Output of the REST API endpoint for your newly deployed API:

```
aws cloudformation describe-stacks --stack-name MythicalMysfitsServiceStack
```

Below is an example output you should find in the response to the above command (the region listed will match the region you are using):

```
{
    "Description": "The endpoint for the REST API created with API Gateway",
    "OutputKey": "ApiEndpoint",
    "OutputValue": "https://abcde12345.execute-api.us-east-1.amazonaws.com/prod"
}
```

Copy the OutputValue to be used in the following step.
Open the new version of the Mythical Mysfits index.html file we will push to S3 shortly, it is located at: **~/environment/modern-application-workshop/module-4/app/web/index.html**
In this new index.html file, you'll notice additional HTML and JavaScript code that is being used to add a user registration and login experience.  This code is interacting with the AWS Cognito JavaScript SDK to help manage registration, authentication, and authorization to all of the API calls that require it.

In this file, replace the string REPLACE_ME'inside the single quotes with the endpoint OutputValue you copied from above and save the file:


Now, lets copy this file up to the S3 bucket hosting our Mythical Mysfits website so that our new page is available on the Internet:

```
aws s3 cp ~/environment/modern-application-workshop/module-4/web/index.html s3://YOUR-S3-BUCKET/index.html
```

Refresh the Mythical Mysfits website in your browser to see the new functionality in action!

This concludes Module 4.
