# Automated-Historian-IoT-Data-Exploration
IoT data often comes with high temporal resolution (e.g. 1 reading per x seconds), and the underlining machine status
often change continuously with the time progression. It is important to train and update the original ML model on a regular basis
to achieve more accurate data analytics, and avoid model drifting. However in the past, such model re-training process contain several 
manual steps, e.g. data exploration, that are difficult to automate, and time-consuming. They require non-negligible human and capital expense.
This github repository aims to provide a fully managed AWS serivce solution to automate the IoT data exploration process. 
The purpose of automated IoT data exploration is to identify anomalies, outliers, missing values and skewed target distribution, so that it can help the Data Scientist to develop a suitable data preprocessing. 
We envisage that such automated and code-free data exploration will be instrumental to efficient data normalization and accurate ML model.

This is a project using multiple AWS services to automate historian IoT data Exploration. To show case this solution, the input historian data leveraged an open source data set from Kaggle (https://www.kaggle.com/c/ashrae-energy-prediction
). 

## The high-level solution architect for the automated historian IoT Data Exploration and ML

![alt text](https://github.com/Julia-Bobo-Hu/Automated-Historian-IoT-Data-Exploration/blob/master/images/Step1_architecture_revised_batch_method.png?raw=true)

This solution architect is made of three steps. Step 1: Efficient historian data ingestion with two seperate lambda functions and Kinesis stream. Step 2: Multi-resolution datasets through IoT Analytics platform. Step3: Automated data exploration by using IoT Analytics datasets and QuickSight.  

### Step 1, Data Ingestion Pipeline for Historian data
To regularly ingest high-resolution IoT historian dataset for data exploration, Kinesis stream with lambda functions were choosen to ingest data to IoT Anlytics platform. 
Since large amount of historian IoT data is ingested from S3, the IoT Analytics's BatchPutMessage mode is used, and Kinesis's parallel lambda function invocation is also applied to accelerate data ingestion.
The yaml file in folder "cloudformation" is used to automatically setup provisioned services with proper roles and policies for data ingestion workflow. 

One Lambda function, “the launcher”, will iterate through our bucket and upload each IoT datafile object key to Kinesis. 
Kinesis will then trigger the second Lambda function, as "worker". "worker" lambda will download the data located at that S3 key and send BatchPutMessage requests containing the data to IoT Analytics Channel. 
If it encounters an error while doing so, it will be invoked again. Please note, due to the historian datafile size, light-weight and cost-efficient lambda function is used to fetch such datafile, not the Kinesis stream itself.
Such solution will ensure the IoT data storage will only be charged once at IoT Analytics, but not Kinesis stream. 

Deployment packages contain the code for the Lambda functions. We use deployment packages because they allow us to upload dependencies along with the the code. 
This S3 folder contains those packages. The function definitions are displayed below.

### Data Ingestion Figure 1
![alt text](https://github.com/Julia-Bobo-Hu/Automated-Historian-IoT-Data-Exploration/blob/master/images/step1.PNG?raw=true)

To trigger the lambda function in this demo, you’ll need to install the AWS Command Line Interface (CLI) and have Admin role authorization.  
For the initial set up of the CLI on your device please follow these instructions (https://aws.amazon.com/cli/). Additionally, the reader should be able to set up IAM credentials. 
We use the CLI to invocate the launcher lambda function and specify the Kinesis stream as: 
aws lambda invoke --function-name REPLACE_WITH_EXAMPLE_FUNCTION_NAME --payload file://lambdaPayload.json --region us-east-1 lambdaOutput.txt

Due to the large historian datafile size, we made following changes to previous example of S3 data ingestion with Kinesis(https://aws.amazon.com/blogs/iot/ingesting-data-from-s3-by-using-batchputmessage-aws-lambda-and-amazon-kinesis/): 
both Cloudformation for Kinesis streaming provision and "Worker" lambda function have been modified to accomodate parallel lambda invocation. 
Cloudformation for Kinesis stream specify the number of shards as 3. Such modification allows the worker Lambda functions to run concurrently. 
In the "worker" lambda function, the maximum allowed concurrent invocations need to be increased. 
Lastly, you would need to divide the MAX_REQUESTS_PER_SECOND value in the worker Lambda function by the value you assigned to ReservedConcurrentExecutions.

Once the data ingestion process starts, you can monitor the process by checking the log group for "Worker" Lambda function.
When the log stream for "Worker" lambda stopped updating, the data ingestion is finished. The whole process for ingesting 1.8 millon records takes about 12 minutes. 

### Step 2, Dataset Generation and Storage with IoT Analytics

After data ingested with worker lambda function, the data will be passed on to IoT Analytics for further process and automation. A quick overview of the IoT Analytics services used in this repository (Figure 2):  
Channels ingest data, back it up, and publish it to one or more pipelines.
Pipelines ingest data from a channel and allow you to process the data through activities before storing it in a data store.
Data stores store data. They are scalable and queryable.
Datasets retrieve data from a datastore. They are the result of some SQL query run against the data store.

Another changes made to the current cloudformation template, the provision of IoT Analytics has also be provided in a "infractures as software" manner. 
Suitable services role for Lambda function to BatuchPutMessage to IoT Analytics has been provisioned by this CloudFormation template as well.
This will improve the consistancy of the AWS service provision.  

### IoT Analytics Step2 figure 
![alt text](https://github.com/Julia-Bobo-Hu/Automated-Historian-IoT-Data-Exploration/blob/master/images/step2.PNG?raw=true)

### Step 3, Create low temporal resolution Dataset for QuickSight SPICE data ingestion 

Two datasets were generated from the IoT Datastore. The high-resolution dataset, which selects every record within the Datastore, is generated for future Sagemaker Notebook model training.
The low temporal resolution dataset aggreates the timestamp from minutes to days, then calculates average meter reading, and multiple weather readings (e.g. air temperature, dew temperature, wind speed, wind direction, cloud coverage, etc.). 
This SQL query also select the first time-related record within each time feature after groupby subfunction, to reduce the temporal resolution. 
The sql_query folder contains relavant sql queries used to generate two seperate IoT Datasets.

### Step 4, Several examples of using QuickSight dashboard to achieve code-free data exploration are explained. 

In this repository, the low resolution IoT Analytics dataset is imported to Quicksight for dashboard. 
On QuickSight, click on New analysis, followed by New data set. Next, click on AWS IoT Analytics, then import the dataset as SPICE dataset in QuickSight. 
After importing the dataset, you can click on "edit" to refine the dataset. A few common operations are: defining timestamp column as datetime type. 
Define relational hiarachy for "country", "state" and "city" columns for drill down functions. In this file, three examples are explained to show how to achieve data exploration in a code-free way by using QuickSight. 

#### Example 1: Use drill-down function for building usage exploration

![alt text](https://github.com/Julia-Bobo-Hu/Automated-Historian-IoT-Data-Exploration/blob/master/images/quicksight_example1.PNG?raw=true)

First, select visual type as "map", Use "country-state-city"hiarachy data for geospacial dimention, then use "building_id(count)" for size, and this will illustrate how many buildings for each city.
Last use "primary use" for color, and it can show different building usages by colors. Enduser can drill down to different granuality level by clicking "drill down" function. 

#### Example 2: Plot multiple columns together for missing value exploration
![alt text](https://github.com/Julia-Bobo-Hu/Automated-Historian-IoT-Data-Exploration/blob/master/images/quicksight_example2.PNG?raw=true)

In this example, the bar chart is chosen as visual type. X_axis is chosen "timestamp_new", and Y_axis is chosen as multiple columns, 
e.g. avg_air_temp (count), avg_cloud_cov(count), avg_precip(count), avg_dew_temp(count). 
From the comparison, it can be seen that "avg_cloud_cov(count)" missed half of the values.

#### Example 3: Detect outliers with Treemap
![alt text](https://github.com/Julia-Bobo-Hu/Automated-Historian-IoT-Data-Exploration/blob/master/images/quicksight_example3.PNG?raw=true)

Select visual type as "Tree map", then select "building_id" as groupby column, then select "avg_meter_reading(average)" for size, then "avg_meter_reading(average)" for color.
To detect outliers only for meter type: cold steam, setup a filter, use meter_type == cold steam as filter criteria to constrain outlier detection for one meter type.
It can be seen that building_id == 1099 has significant higher meter reading values compared with all other buildings. Data Scientist can use this method for outlier removal.

#### Example 4: Detect excessive empty values 
![alt text](https://github.com/Julia-Bobo-Hu/Automated-Historian-IoT-Data-Exploration/blob/master/images/quicksight_example4.PNG?raw=true)

Select visual type as "Heatmap", then select "timestamp_new" as Row, then select "site_id" as Columns, last, choose "avg_meter_reading(count)" as Values.
To detect large number of missing values, setup a filter, use meter_reading == 0 as filter criteria to constrain counts to these missing values only.
It can be seen that site0 has significant higher missing values in the first 6 months compared with all other sites. Data Scientist can also use this method for abnomal values removal.
  



