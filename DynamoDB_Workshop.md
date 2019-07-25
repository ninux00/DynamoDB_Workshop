# DynamoDB Practical Workshop
July, 2019 Version

Overview
This lab document is meant to provide some practical exercises of implementing design patters using Amazon DynamoDB. Here's some of what you'll find:
List of exercises:

*	Preparation
*	Exercise 1 - Capacity Units & Partitioning
*	Exercise 2 - Table Scan and Parallel Scan
*	Exercise 3 - GSI write sharding
*	Exercise 4 - GSI key overloading
*	Exercise 5 - Sparse Indexes
*	Exercise 6 - Composite keys
*	Exercise 7 - Transaction locking
*	Exercise 8 - DynamoDB Streams and AWS Lambda

# Who is it for?
*	Developers looking for recommendations
*	Database professionals looking for NoSQL and DynamoDB details
# Requirements
*	Basic experience with AWS
*	Basic knowledge on DynamoDB
*	Some experience with Python

# Preparation 
## Step 1 - Deploy the Cloudformation template.

Click on the "Deploy" link below that references the region assigned to you.

Region| Deploy
------|-----
US East 1 (N.Virginia) | <a href="https://console.aws.amazon.com/cloudformation/home?region=us-east-1#/stacks/new?stackName=DynamoDB-Workshop&templateURL=https://dynamodb-workshop.s3.amazonaws.com/cloudformation_cloud9.json" target="_blank">![Deploy in us-east-1](./images/deploy-to-aws.png)</a>
US East 2 (Ohio) | <a href="https://console.aws.amazon.com/cloudformation/home?region=us-east-2#/stacks/new?stackName=DynamoDB-Workshop&templateURL=https://dynamodb-workshop.s3.amazonaws.com/cloudformation_cloud9.json" target="_blank">![Deploy in us-east-2](./images/deploy-to-aws.png)</a>
US West 2 (Oregon) | <a href="https://console.aws.amazon.com/cloudformation/home?region=us-west-2#/stacks/new?stackName=DynamoDB-Workshop&templateURL=https://dynamodb-workshop.s3.amazonaws.com/cloudformation_cloud9.json" target="_blank">![Deploy in us-west-2](./images/deploy-to-aws.png)</a>

On the CloudFormation wizard just follow clicking on "Next" button until you have the option to click on "Create Stack" button and wait to the creation of the stack.

Once the stack is created, navigate to the Cloud9 console as the image below.

