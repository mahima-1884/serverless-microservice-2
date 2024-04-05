# serverless-microservice-2

# Serverless Application with REST API – Part 1

## 1. Create the SQS queue

1. Create an SQS queue
2. Use the standard queue type
3. Name it: `ProductOrdersQueue`
4. Copy the queue URL and note it somewhere

## 2. Create the first Lambda function

1. Create a Lambda function with the following settings
- Name: SubmitOrderFunction
- Python 3.9 runtime

2. Add the following code. Replace the `YOU_SQS_URL` with your SQS URL
   ```python
     import json
    import boto3

    sqs = boto3.client('sqs')
    queue_url = 'YOUR_SQS_URL'

    def lambda_handler(event, context):
      try:
        order_details = json.loads(event['body'])
        response = sqs.send_message(QueueUrl=queue_url, MessageBody=json.dumps(order_details))
        return {'statusCode': 200, 'headers': {'Access-Control-Allow-Origin': '*', 'Access-Control-Allow-Headers': 'Content-Type', 'Access-Control-Allow-Methods': 'OPTIONS,POST'}, 'body': json.dumps({'message': 'Order submitted to queue successfully'})}
      except Exception as e:
        return {'statusCode': 400, 'body': json.dumps({'error': str(e)})}
        ```
3. Go to IAM Roles. Select `SubmitOrderFunctionRolexxxxx`
4. Add the `AmazonSQSFullAccess` permissions policy to the role
5. Deploy the code.
6. Create and submit a test event with the following data

```json
{
  "body": "{\"productName\":\"Test Product\",\"quantity\":3}"
}
```

7. Go to SQS and poll for messages. A message will be waiting in the queue

## 3. Test order submissions through CLI

1. Open CloudShell
3. Type `nano payload.json`
4. Inside the editor. Copy paste the following command
```
   {
  "body": "{\"productName\":\"Test Product\",\"quantity\":3}"
}
```
5. Exit the editor
6. Type the following command in CloudShell
```bash
aws invoke lambda --function-name SubmitOrderFunction --payload fileb://payload.json response.json
```
7. Type `ls`. Note a new file called `response.json` is created
8. Open `response.json`. A status code : 200 indicates successful invoke of Lambda function
9. Check the SQS queue. 2 messages should appear in waiting.

## 4. Create the DynamoDB table

1. Create a DynamoDB table with the following settings
- Name: ProductOrders
- Primary key: orderId

## 5. Create the processing function

1. Create a Lambda function with the following settings
- Name: ProcessOrderFunction
- Python 3.9 runtime

2. Add the following code
```python
  import json
import boto3
from datetime import datetime

dynamodb = boto3.resource('dynamodb')
table_name = 'ProductOrders'
table = dynamodb.Table(table_name)

def lambda_handler(event, context):
    for record in event['Records']:
        message_body = json.loads(record['body'])
        item = {'orderId': record['messageId'], 'productName': message_body['productName'], 'quantity': message_body['quantity'], 'orderDate': datetime.now().isoformat()}
        table.put_item(Item=item)
    return {'statusCode': 200, 'body': json.dumps({'message': 'Orders processed successfully'})}
```
3. Add Lambda Trigger. Search service `SQS` and select `ProductOrdersQueue` from the dropdown list
3. Go to IAM roles and select `ProcessOrderFunctionRolexxxxxx` execution role
4. Add the `AmazonSQSFullAccess` and `AmazonDynamoDBFullAccess` permissions policies to the role

## 6. Deploy and test the application

1. Check the DynamoDB table to see if the first test event was processed
2. Test using the CLI. Using CloudShell create a file named `input.json` with the following contents

```json
{
  "body": "{\"productName\":\"Test Product 2\",\"quantity\":2}"
}
```

3. Invoke the function:

```bash
aws lambda invoke --function-name <function-name> --payload fileb://input.json output.json
```

## 7. Create the API

1. Create a REST API in the API Gateway Console named `ProductOrdersAPI`
2. Create a new resource `/orders` and enable CORS
3. Create a `POST` method for `/orders` integrated with the `SubmitOrderFunction`
4. Enable a Lambda proxy integration
5. Click back up to the `/orders` resource and click "Enable CORS"
6. Select all CORS options
7. Click `Deploy`. Add a new stage `Prod`
8. Update the `POST` method invoke URL in the 'order.html' file on line 32 where it says `YOUR_API_ENDPOINT`. 

***note, the invoke URL on line 32 should include /prod/orders on the end and look like this example***

'https://zzo3imx0a4.execute-api.eu-west-1.amazonaws.com/Prod/orders'

## 8. Create the static website bucket and test the application

1. In Amazon S3 create a bucket
2. Configure the bucket for static website hosting
3. Set the default document to `order.html`
4. Enable public access using a bucket policy

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "PublicReadGetObject",
            "Effect": "Allow",
            "Principal": "*",
            "Action": "s3:GetObject",
            "Resource": "<YOUR-BUCKET-ARN>/*"
        }
    ]
}
```

5. Upload the edited 'order.html' that has the API endpoint URL configured
6. Navigate to the static website endpoint
7. Submit some order information
8. An "Order submitted successfully!" response will appear
9. Check DynamoDB table for updated order information





