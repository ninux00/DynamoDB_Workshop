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

![AWS Console](./images/aws_console.png)

It will bring you to the Cloud9 console where you will find an Environment named "DynamoDB-Workshop-Environmet", click on the "Open IDE" button.

![Clou9 Console](./images/cloud9_console.png)

Wait a few moments and you will be redirected to the IDE of your new environment.

At the botton of the screen you have access to a terminal.
In this workshop you will use this terminal to run all the commands. 
Few free to explore the IDE.

![Clou9 IDE](./images/cloud9_ide.png)

## Step 2 – Download and Check the content of the workshop folder

Download content:

```
aws s3 cp s3://hugorozestraten.net/dynamodbws/workshop.zip .
mkdir workshop
cd workshop
unzip ../workshop.zip
```

Go to the workshop folder and run the ls command:

```
cd /home/ec2-user/workshop
ls -l .
```

You will see the following content:

#### Python code:

*	ddbreplica_lambda.py
*	load_employees.py
*	load_tlog_parallel.py
*	load_tlog.py
*	query_city_dept.py
*	query_employees.py
*	query_managers_gsi.py
*	query_managers_table.py
*	scan_tlog_parallel.py
*	scan_tlog_simple.py
*	update_employees.py

#### JSON:

*	gsi_city_dept.json
*	gsi_manager.json
*	iam-role-policy.json
*	iam-trust-relationship.json

Run the ls command to show the sample data files:

```
ls -l ./data
```

#### ./data

*	employees.csv
*	logfile_medium1.csv
*	logfile_medium2.csv
*	logfile_small1.csv
*	logfile_stream.csv


## Step 3 - Check the files format and content

You will be working with two different data contents during this lab: (1)Server logging data and (2)Employees data.
The log files have the following structure:

*	requestid (number)
*	host (string)
*	date (string)
*	hourofday (number)
*	timezone (string)
*	method (string)
*	url (string)
*	responsecode (number)
*	bytessent (number)
*	useragent (string)

To view a sample record in the file, execute:

```
head ./data/logfile_small1.csv -n 1
```

Sample log record:

```1,66.249.67.3,2017-07-20,20,GMT-0700,GET,"/gallery/main.php?g2_controller=exif.SwitchDetailMode&g2_mode=detailed&g2_return=%2Fgallery%2Fmain.php%3Fg2_itemId%3D15741&g2_returnName=photo",302,5,"Mozilla/5.0 (compatible; Googlebot/2.1; +http://www.google.com/bot.html)"```

The employees files have the following structure:

*	employeeid (number)
*	name (string)
*	title (string)
*	dept (string)
*	city (string)
*	state (string)
*	dob (string)
*	hire_date (string)
*	previous title (string)
*	previous title end date (string)
*	is a manager (string), 1 for manager employees, non-existent for others

To view a sample record in the file, execute:

```
head ./data/employees.csv -n 1
```
Sample employee record:

```
1,Onfroi Greeno,Systems Administrator,Operation,Portland,OR,1992-03-31,2014-10-24,Application Support Analyst,2014-04-12
```

## Step 4 - Preload the items for the scan exercise

In the exercise 2 we will discuss table scan and the alternatives. In this step, you are going to load the table with 1 milliton rows in preparation for the exercise.
Run the command to create a new table:

```
aws dynamodb create-table --table-name tlog_scan \
--attribute-definitions AttributeName=requestid,AttributeType=N AttributeName=gsi_responsecode_hk,AttributeType=N AttributeName=gsi_responsecode_sk,AttributeType=S \
--key-schema AttributeName=requestid,KeyType=HASH \
--provisioned-throughput ReadCapacityUnits=5000,WriteCapacityUnits=5000 \
--global-secondary-indexes  IndexName=gsi_responsecode,\
KeySchema=["{AttributeName=gsi_responsecode_hk,KeyType=HASH},{AttributeName=gsi_responsecode_sk,KeyType=RANGE}"],\
Projection="{ProjectionType=KEYS_ONLY}",\
ProvisionedThroughput="{ReadCapacityUnits=3000,WriteCapacityUnits=5000}"
```

This command will create a new table and one GSI with the following definition:

Table: tlog_scan

*	Attribute: requestid
*	Key Type: Hash
*	Table RCU = 5000
*	Table WCU = 5000

Run the command to wait until the table becomes Active:

```
aws dynamodb wait table-exists --table-name tlog_scan
```

Populate the table

Run the following command to load the server logs data into the tlog_scan table. It will load 78688 rows to the table.

```
python load_tlog_parallel.py tlog_scan
```
```total rows 1000000 in 514.663530111 seconds```

# Exercise 1: DynamoDB Capacity Units & Partitioning
In this exercise you will load data into DynamoDB tables that are provisioned with different Write/Read capacity units, and compare the load times. You will be using the “log” data.

##### Open a Second terminal in your Cloud9 IDE by clicking on the "+" icon next to the Terminal's tab.
 
#### Step 1 – Create the DynamoDB table
Run the following AWS CLI command to create the first DynamoDB table called 'tlog':
```
aws dynamodb create-table --table-name tlog \
--attribute-definitions AttributeName=requestid,AttributeType=N AttributeName=host,AttributeType=S \
--key-schema AttributeName=requestid,KeyType=HASH \
--provisioned-throughput ReadCapacityUnits=5,WriteCapacityUnits=5 \
--global-secondary-indexes  IndexName=host-requestid-gsi,\
KeySchema=["{AttributeName=host,KeyType=HASH},{AttributeName=requestid,KeyType=RANGE}"],\
Projection="{ProjectionType=INCLUDE,NonKeyAttributes=['bytessent']}",\
ProvisionedThroughput="{ReadCapacityUnits=5,WriteCapacityUnits=5}"
```

The table you just created will have the following structure.

Table:
*	Name: tlog
*	Partition Key: requestid
*	Table RCU = 5
*	Table WCU = 5

GSI:
*	Name: host-requestid-gsi
*	Partition Key: host
*	Sort Key: requestid
*	Projection Include: bytessent
*	GSI RCU = 5
*	GSI WCU = 5

Run the command to wait until the table becomes Active:

```
aws dynamodb wait table-exists --table-name tlog
```

You can run the following command to get only the table status:

```
aws dynamodb describe-table --table-name tlog | grep TableStatus
```

#### Step 2 - Load sample data into the table

Now that you have the table created, you can load some sample data running the Python script:

```
python load_tlog.py tlog ./data/logfile_small1.csv
```

Parameters in the above command:
1) Table Name = tlog
2) File Name = logfile_small1.csv

The output should be like:

``` 
row: 100 in 0.790903091431
row: 200 in 0.746693134308
row: 300 in 0.797533035278
row: 400 in 0.695877075195
row: 500 in 0.715097904205 
RowCount: 500, Total seconds: 3.79715394974
```

#### Step 3 - Load a larger file to compare the execution time

Run the script again but at this time with a larger input data file:

```
python load_tlog.py tlog ./data/logfile_medium1.csv
```

Parameters:
1) Table Name = tlog
2) File Name = logfile_medium1.csv

The output should be like:

```
row: 100 in 0.490761995316
row: 200 in 0.449488162994
row: 300 in 0.450911045074
row: 400 in 0.450123071671
row: 500 in 0.445238828659
row: 600 in 0.58230304718
row: 700 in 0.581787824631
row: 800 in 0.595230102539
row: 900 in 0.588124990463
row: 1000 in 0.589934110641
row: 1100 in 0.588397979736
row: 1200 in 5.44571805
row: 1300 in 19.9914531708
row: 1400 in 20.033246994
row: 1500 in 19.9791162014
row: 1600 in 20.0433659554
row: 1700 in 19.9400119781
row: 1800 in 20.002464056
row: 1900 in 19.998513937
row: 2000 in 20.0029530525
RowCount: 2000, Total seconds: 171.298333883
```

#### After loading some rows, the load time increased. In the sample output, the time increased from less than 1 sec to around 20 seconds.

#### Step 4 - View the CloudWatch metrics on your table

Open the browser tab in the AWS console. If you don't have the AWS console opened, see the step 2 of the Preparation section above.

Refresh the page if necessary to be able to see the left menu.

On the left menu, click Tables, on the table click "Metrics".

![Metrics](./images/metrics.png)

You see the metrics like the sample below:

![Metrics1](./images/metrics1.png)

##### Question: Why there are throttling events on the table and there are not on the GSI?

## Step 5 – Increase the capacity

Run the following AWS CLI command to increase the WCU and RCU from 5 to 100.

```
aws dynamodb update-table --table-name tlog \
--provisioned-throughput ReadCapacityUnits=100,WriteCapacityUnits=100
```

Run the command to wait until the table becomes Active.

```
aws dynamodb wait table-exists --table-name tlog
```

##### Question: How long did it take for increasing the capacity?

## Step 6 - Load more data with the new capacity

After the update, run the Python script again to populate the table with file logfile_medium2.csv with the same number of rows of the previous run. You will see that at this time the execution will be faster.

```
python load_tlog.py tlog ./data/logfile_medium2.csv
```

The output should show a result like:

```
row: 100 in 0.678274869919
row: 200 in 0.637241125107
row: 300 in 0.623446941376
row: 400 in 0.666973114014
row: 500 in 0.694450139999
row: 600 in 0.643283843994
row: 700 in 0.644812107086
row: 800 in 0.63000702858
row: 900 in 0.76971411705
row: 1000 in 0.684530973434
row: 1100 in 0.667228937149
row: 1200 in 0.66502404213
row: 1300 in 0.681458950043
row: 1400 in 0.644996166229
row: 1500 in 0.657227993011
row: 1600 in 0.650546073914
row: 1700 in 0.643900156021
row: 1800 in 0.62757897377
row: 1900 in 0.646264076233
row: 2000 in 0.701603889465
RowCount: 2000, Total seconds: 13.3077070713
```
##### Note: With the new capacity the load time was consistent for the whole dataset.

## Step 7 – Low capacity GSI

For the next step, we are going to create a new table with different capacity units. At this time, the GSI will have on 1 WCU and 1 RCU, for the purpose of this exercise.

Run the following command:

```
aws dynamodb create-table --table-name tlog_gsi_low \
--attribute-definitions AttributeName=requestid,AttributeType=N AttributeName=host,AttributeType=S \
--key-schema AttributeName=requestid,KeyType=HASH \
--provisioned-throughput ReadCapacityUnits=1000,WriteCapacityUnits=1000 \
--global-secondary-indexes  IndexName=host-requestid-gsi,\
KeySchema=["{AttributeName=host,KeyType=HASH},{AttributeName=requestid,KeyType=RANGE}"],\
Projection="{ProjectionType=INCLUDE,NonKeyAttributes=['bytessent']}",\
ProvisionedThroughput="{ReadCapacityUnits=1,WriteCapacityUnits=1}"
```

Run the command to wait until the table becomes Active:

```
aws dynamodb wait table-exists --table-name tlog_gsi_low
```

This command will create a new table and one GSI with the following definition:

Table: tlog_gsi_low

*	Attribute: requestid
*	Key Type: Hash
*	Table RCU = 1000
*	Table WCU = 1000

GSI: host-requestid-gsi

*	Attribute: host
*	Key Type: Hash
*	Attribute: requestid
*	Key Type: Range
*	Projection Include: bytessent
*	GSI RCU = 1
*	GSI WCU = 1
*	

You can, now, populate the new table with a large dataset. At this time, you will run a script that uses a multi-thread version to have more writes per second on the DynamoDB table.

```
python load_tlog_parallel.py tlog_gsi_low
```

Eventually, the execution will be throttled and show the error message like below:

```Error:Error: (, ProvisionedThroughputExceededException(u'An error occurred (ProvisionedThroughputExceededException) when calling the PutItem operation (reached max retries: 9): The level of configured provisioned throughput for one or more global secondary indexes of the table was exceeded. Consider increasing your provisioning level for the under-provisioned global secondary indexes with the UpdateTable API',), )```

You can cancel the operation, typing Ctrl+C. It can take some time to cancel.

##### Note: This new table has more RCU=1000 and WCU=1000 but we received an error and the load time increased.

##### Question: Could you explain the behavior of the test?

Open the AWS console to view the metrics for the table tlog_gsi_low.

![Metrics2](./images/metrics2.png)

Note above that the consumed writes were below the provisioned writes for the table during the test.

![Metrics3](./images/metrics3.png)

Note above that the consumed writes were high than the provisioned writes for the GSI.

![Metrics4](./images/metrics4.png)

Note that the table was throttled, even having enough write capacity.

![Metrics5](./images/metrics5.png)

## Exercise 2 - Table Scan and Parallel Table Scan





