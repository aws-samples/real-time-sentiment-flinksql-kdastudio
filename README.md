# Real-time sentiment analysis on customer feedback

## Reference architecture

![kda1](/images/kda1.PNG)

## Create a Kinesis Data Analytics Studio notebook
1. Go to Kinesis Data Analytics Console: console.aws.amazon.com/kinesisanalytics
2. Click on the Studio tab
3. Click on create Studio notebook
4. Choose "Quick create with sample code" as create method
5. Enter a notebook name
6. For AWS Gluedatabse click on the refresh button and select Default Glue database. If the list is still empty, create a new Glue database.
7. Note down the IAM role name. Click on Create Studio Notebook.

![kda1](/images/kda2.png)


## Configure IAM
1. Go to IAM: https://console.aws.amazon.com/iam
2. Click on the role and search for the role that KDA Studio has created earlier
3. Click on Attach Policies and add Administrator Access. (This is not recommended for your production workload.)

![kda1](/images/kda3.png)

## Working with Kinesis Data Analytics Studio
1. Go to Kinesis Data Analytics Console: console.aws.amazon.com/kinesisanalytics
2. Click on the Studio tab and select the notebook you have created in the previous step
3. Click on Run and Click on Open in Apache Zeppelin once the statue of the Notebook is running
![kda4](/images/kda4.png)

## Working with Kinesis Data Analytics Studio - create a prerequisite notebook
1. In the notebook console create a new note
2. enter the name of your notebook- "prerequisite"
3. Select Default Interprete as Flink and create the notebook
![kda5](/images/kda5.png)

** We are going to use the notebook to provisioned some AWS resources, for example, DynamoDB table, Kinesis Data Streams etc. For that, we are using boto3. 

4. The boto3 library is preinstalled. Run the below command if step 5 is showing an error on boto3. Execute the following code to install boto3. 
```
%flink.ipyflink

pip install boto3
```

5. Create a new paragraph and execute the below code. This will create a DynamoDB table in us-east-1
```
%flink.ipyflink
#create table innovate_latlon
import boto3
region='us-east-1'
dynamodb = boto3.resource('dynamodb',region_name=region)
response = dynamodb.create_table(
    AttributeDefinitions=[
        {
            'AttributeName': 'pk',
            'AttributeType': 'S'
        },
    ],
    TableName='innovate_latlon',
    KeySchema=[
        {
            'AttributeName': 'pk',
            'KeyType': 'HASH'
        },
    ],
    BillingMode='PAY_PER_REQUEST'
)
```
![kda6](/images/kda6.png)

6. Create a new paragraph and execute the below code. This will create another DynamoDB table in us-east-1
```
%flink.ipyflink
#create table innovate_custfeedback
import boto3
region='us-east-1'
dynamodb = boto3.resource('dynamodb',region_name=region)
response = dynamodb.create_table(
    AttributeDefinitions=[
        {
            'AttributeName': 'pk',
            'AttributeType': 'S'
        },
    ],
    TableName='innovate_custfeedback',
    KeySchema=[
        {
            'AttributeName': 'pk',
            'KeyType': 'HASH'
        },
    ],
    BillingMode='PAY_PER_REQUEST'
)
```

7. Create a new paragraph and execute the below code. This will create a Kinesis data stream in us-east-1
```
%flink.ipyflink
#create KDS innovate_feedback
import boto3
region='us-east-1'
kinesis = boto3.client('kinesis',region_name=region)
response = kinesis.create_stream(
    StreamName='innovate_feedback',
    ShardCount=3
)
print (response)

```

## Uploading sample data
1. Create a new S3 bucket or upload the below CSV files to your S3 bucket

    a) [custfeedback.csv](sampledata/custfeedback.csv)

    b) [latlon.csv](sampledata/latlon.csv)
 
 2. Create a new paragraph on your prerequisite notebook and execute the below code. Change the S3 location as your's (bucket, key). This will upload the latlon data to a DynamoDB table you created earlier.
 
 ```
 %flink.ipyflink
#upload lanlon data
import boto3
import csv
import codecs
region='us-east-1'
recList=[]
tableName='innovate_latlon'
s3 = boto3.resource('s3')
dynamodb = boto3.client('dynamodb', region_name=region)
bucket='YOUR_BUCKETNAME'
key='real-time-sentiment-flinkSQL-KDAStudio/latlon.csv'
obj = s3.Object(bucket, key).get()['Body']
batch_size = 100
batch = []
i=0

for row in csv.DictReader(codecs.getreader('utf-8')(obj)):
    pk= (row["id"])
    postcode= (row["postcode"])
    suburb= (row["suburb"])
    State= (row["State"])
    latitude= (row["latitude"])
    longitude= (row["longitude"])
    
    response = dynamodb.put_item(
        TableName=tableName,
        Item={
        'pk' : {'S':str(pk)},
        'postcode': {'S':postcode},
        'suburb': {'S':suburb},
        'State': {'S':State},
        'latitude': {'S':latitude},
        'longitude': {'S':longitude}
        }
    )
    i=i+1
    #print ('Total insert: '+ str(i))
    
print ('completed')
 ```

3. Create a new paragraph on your prerequisite notebook and execute the below code. Change the S3 location as your's (bucket, key). This will upload the customer feedback data to a DynamoDB table you created earlier.

```
%flink.ipyflink
#upload custfeedback.csv
import boto3
import csv
import codecs
region='us-east-1'
recList=[]
tableName='innovate_custfeedback'
s3 = boto3.resource('s3')
dynamodb = boto3.client('dynamodb', region_name=region)
bucket='YOUR_BUCKETNAME'
key='real-time-sentiment-flinkSQL-KDAStudio/custfeedback.csv'
obj = s3.Object(bucket, key).get()['Body']
batch_size = 100
batch = []
i=0

for row in csv.DictReader(codecs.getreader('utf-8')(obj)):
    pk= (row["id"])
    feedback= (row["feedback"])
    
    response = dynamodb.put_item(
        TableName=tableName,
        Item={
        'pk' : {'S':str(pk)},
        'feedback': {'S':feedback}
        }
    )
    i=i+1
    #print ('Total insert: '+ str(i))
    
print ('completed:' + str(i))
```

## Generating random data to Kinesis data streams
1. Create a new paragraph on your prerequisite notebook and execute the below code. This will analyze customer feedback sentiment using Amazon comprehend and send those data (30,000 ingestions) to a Kinesis Data Stream.

```
%flink.ipyflink
#KDS random data generator
import json
import boto3
import csv
import datetime
import random
from boto3.dynamodb.conditions import Key
tablename_latlon='innovate_latlon'
kdsname='innovate_feedback'
tablename_innovate_custfeedback='innovate_custfeedback'
region='us-east-1'
i=0
clientkinesis = boto3.client('kinesis',region_name=region)



def getfeedback():
    dynamodb = boto3.resource('dynamodb',region_name=region)
    table = dynamodb.Table(tablename_innovate_custfeedback)
    randomnum = random.randint(1, 1000)
    response = table.query(
        KeyConditionExpression=Key('pk').eq(str(randomnum))
    )
    items=response['Items']
    for item in items:
        feedback=item['feedback']
    return feedback

def getproduct(i):
    product=["Kindle 10th Gen", "Fire TV Stick Lite", "Amazon eero mesh", "Ring Home Security", "Echo Dot 4th Gen", 'Roborock Vacuum cleaners','Panasonic LUMIX Camera', 'Philips Air Purifier', 'Anker Chargers hub']
    return (product[i])

def getsentiment(mystr):
    client = boto3.client('comprehend',region_name=region)
    response = client.detect_sentiment(Text=mystr, LanguageCode='en')
    return response['Sentiment']
    
def getlanlon():
    dynamodb = boto3.resource('dynamodb',region_name=region)
    table = dynamodb.Table(tablename_latlon)
    randomnum = random.randint(1, 16491)
    response = table.query(
        KeyConditionExpression=Key('pk').eq(str(randomnum))
    )
    items=response['Items']
    #lat='222'
    #lon='123'
    for item in items:
        lat=item['latitude']
        lon=item['longitude']
        state=item['State']
        postcode=item['postcode']
        suburb=item['suburb']
    return lat, lon, state, postcode, suburb



for x in range(30000):
    i=int(i)+1
    product=getproduct(random.randint(0, 8))
    event_time=datetime.datetime.now().isoformat()
    lat, lon,state,postcode, suburb=getlanlon()
    feedback=getfeedback()
    sentiment=getsentiment(feedback)
    new_dict={}
    new_dict["product"]=product
    new_dict["sentiment"]=sentiment
    new_dict["feedback"]=feedback
    new_dict["event_time"]=event_time
    new_dict["lat"]=lat
    new_dict["lon"]=lon
    new_dict["state"]=state
    new_dict["postcode"]=postcode
    new_dict["suburb"]=suburb
    clientkinesis.put_record(
                    StreamName=kdsname,
                    Data=json.dumps(new_dict),
                    PartitionKey=product)
    
print('###total rows:#### '+ str(i))
```

## Real-time analytics with Flink SQL
1. In the notebook console create a new note
2. enter the name of your notebook- "flinkSQLExample"
3. Select Default Interprete as Flink and create the notebook
4. Execute the below code

```
%flink.ssql

CREATE TABLE innovate_feedback (
    product VARCHAR(50),
    sentiment VARCHAR(50),
    feedback VARCHAR(500),
    lat DOUBLE,
    lon DOUBLE,
    state VARCHAR(20),
    postcode VARCHAR(30),
    suburb VARCHAR(30),
    event_time TIMESTAMP(3),
    WATERMARK FOR event_time AS event_time - INTERVAL '5' SECOND
)
PARTITIONED BY (product)
WITH (
    'connector' = 'kinesis',
    'stream' = 'innovate_feedback',
    'aws.region' = 'us-east-1',
    'scan.stream.initpos' = 'LATEST',
    'format' = 'json',
    'json.timestamp-format.standard' = 'ISO-8601'
)

```
5. Add a new paragraph and start analyzing data in real-time
```
%flink.ssql(type=update)

SELECT * FROM innovate_feedback;

```
![kda7](/images/kda7.png)

6.  Add a new paragraph and execute the below code. Stop the Flink job once you see the table with the column name. Click on Settings, change the visualization as highlighted below and execute the code again.
 
```
%flink.ssql(type=update)
--Product wise sentiment
SELECT innovate_feedback.product, COUNT(*) AS totalsentiment, innovate_feedback.sentiment,
       TUMBLE_END(event_time, INTERVAL '10' second) as tum_time
  FROM innovate_feedback
GROUP BY TUMBLE(event_time, INTERVAL '10' second), innovate_feedback.product, innovate_feedback.sentiment;

```
![kda8](/images/kda8.png)

7. Add a new paragraph and execute the below code. Stop the Flink job once you see the table with the column name. Click on Settings, change the visualization as highlighted below and execute the code again.
 
```
%flink.ssql(type=update)
--state wise sentiment
SELECT innovate_feedback.state, COUNT(*) AS totalsentiment, innovate_feedback.sentiment,
       TUMBLE_END(event_time, INTERVAL '10' second) as tum_time
  FROM innovate_feedback
GROUP BY TUMBLE(event_time, INTERVAL '10' second), innovate_feedback.state, innovate_feedback.sentiment;

```
![kda9](/images/kda9.png)
