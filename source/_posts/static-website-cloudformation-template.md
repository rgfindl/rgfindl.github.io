---
title: Static Website CloudFormation Template
date: 2017-08-07 09:02:12
foreword: Create a static website on AWS with the click of a button.
tags:
---
The template: {% raw %}<a href="https://s3.amazonaws.com/thestackshack/static.website.cloudformation.json">static.website.cloudformation.json</a>{% endraw %}
### Click here to launch your static website on AWS
{% raw %}<a href="https://console.aws.amazon.com/cloudformation/home?region=us-east-1#/stacks/new?stackName=staticwebsite&templateURL=https://s3.amazonaws.com/thestackshack/static.website.cloudformation.json"><img src="https://s3.amazonaws.com/cloudformation-examples/cloudformation-launch-stack.png"/></a>{% endraw %}

### What is a static website and why are they cool?
A static website is simply html, css, js,... files hosted by AWS S3.  They are cool because there are no servers for you to maintain and pay for.  You only pay for the storage of your files and the amount of traffic your site gets.  Security is greatly improved because your site has no servers or databases to hack into.

### What AWS resources does this template use?
* S3 (File storage & static website)
* CloudFront (HTTPS & caching)
* Certificate Manager (SSL Cert)
* Route53 (DNS)
* CodeBuild (Continuous deployment)
* CodePipeline (Continuous deployment)
* CodeCommit (GIT repo)
* CloudFormation (Infrastructure as Code)
* IAM (AWS permissions & users)

This seems like a lot just to host a static website!  Yep, it is, that's why I created this handy CloudFormation template so the next time I want to stand up a _simple_ static website I just have to click the button above.

### How do I get started?
#### Step 1 - Create your AWS account
The first step is to create an {% raw %}<a href="https://aws.amazon.com">AWS account</a>{% endraw %}.

#### Step 2 - Create your IAM user
Once you're logged into your account you need to create an {% raw %}<a href="https://console.aws.amazon.com/iam/home">IAM user</a>{% endraw %} with **Programmatic access**, and with **AWSCodeCommitFullAccess** & **AdministratorAccess** permissions.  Also, your user will need CodeCommit (GIT) credentials.  Don't forget to download all of your credentials because you'll need then later.  See the screen shots below:
{% raw %}<img src="/images/aws-permissions.png"/>{% endraw %}
{% raw %}<img src="/images/aws-git-credentials.png"/>{% endraw %}

#### Step 3 - Purchase your domain name
Head on over to {% raw %}<a href="https://console.aws.amazon.com/route53/home">Route 53</a>{% endraw %} and register your domain name.  If you already have a domain name, don't worry, you can point it at Route53 after you build the stack. 

#### Step 4 - Build the stack
Click on the buttom at the top to build the stack.  Use your domain name and app name as the input parameters.

You can also copy the cloudformation template and install the stack using the {% raw %}<a href="http://docs.aws.amazon.com/cli/latest/userguide/installing.html">AWS command-line tools</a>{% endraw %}.  Harder but recommended.  You should really version control your IaC template as well as your code.  Here are the commands to create, update, and delete the stack.

**Create Stack**
```
aws cloudformation create-stack \
--stack-name <example-com> \
--template-body file:///<abs_path>/static.website.cloudformation.json \
--parameters ParameterKey=ApexDomainName,ParameterValue=<example.com> ParameterKey=AppName,ParameterValue=<example> \
--tags=Key=app,Value=<example.com> \
--capabilities CAPABILITY_IAM \
--profile <profile>
```

**Delete Stack**
```
aws cloudformation delete-stack \
--stack-name <example-com> \
--profile <profile>
```

**Update Stack**
```
aws cloudformation update-stack \
--stack-name <example-com> \
--template-body file:///<abs_path>/static.website.cloudformation.json \
--parameters ParameterKey=ApexDomainName,ParameterValue=<example.com> ParameterKey=AppName,ParameterValue=<example> \
--tags=Key=app,Value=<example.com> \
--capabilities CAPABILITY_IAM \
--profile <profile>
```

#### Step 5 - Validate domain ownership
When AWS Certificate Manager creates your SSL certificate it sends and email to the domain name administrator (You).  You have to get that email and click the confirmation link.  Once this step has been completed the stack can continue being built.  More information can be found here:  http://docs.aws.amazon.com/acm/latest/userguide/gs-acm-validate.html

#### Step 6 - Wait for your stack to be built
Check the progress of your stack here:  https://console.aws.amazon.com/cloudformation/home

Once your stack has been built copy the output param which will be your CodeCommit repo URL.  

### What do I do after my stack has been created?
Use your CodeCommit repo and your CodeCommit credentials to start checking in your static site.

CodePipeline and CodeBuild are setup to continuously deploy your site as you push changes to your repo.  

Branches and CodePipeline actions:
**master** -> Pushes changes to APEX <example.com> (yes ssl)
**develop** -> Pushes changes to dev subdomain (no ssl)
