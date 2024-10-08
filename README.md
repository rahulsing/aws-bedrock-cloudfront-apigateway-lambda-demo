# AWS Bedrock App: Using CloudFront APIGateway & Lamdba

## Architecture Overview
This project demonstrates a serverless architecture using AWS services to create an interactive AI chatbot. Here are a few key points about the architecture:


![steamlitapp](https://github.com/rahulsing/aws-bedrock-cloudfront-apigateway-lambda-demo/blob/main/architecture.png?raw=true)

- API Gateway : Acts as the entry point for requests, providing a RESTful API interface.
- Lambda Function : Processes incoming requests from API Gateway and interacts with Amazon Bedrock.
- Amazon Bedrock : Provides the AI/ML capabilities, specifically using the Claude model for natural language processing.
- S3 Bucket : Hosts the static website files (HTML, CSS, JavaScript).
- CloudFront : Serves as a content delivery network (CDN) to distribute the website globally with low latency.

## Prerequisites
Before you begin, ensure you have the following:

- AWS CLI 
- Amazon Bedrock Access : Ensure you have access to Amazon Bedrock in your AWS account. As of 2023, this might require requesting access from AWS.
- IAM Permissions : Your AWS user should have permissions to create and manage the following services:
            - CloudFormation
            - Lambda
            - API Gateway
            - S3
            - CloudFront
            - IAM (for creating roles)
            - Amazon Bedrock

## Demo : 
### Step 1: Clone GitHub Role: 
```
git clone https://github.com/rahulsing/aws-bedrock-cloudfront-apigateway-lambda-demo.git
```

### Step 2: CloudFormation to create API Gateway and Lamdba Function Invoking Bedrock. 

#### Create stack
```
AWS_REGION="us-west-2"
FIRST_STACK=bedrock-api-lambda-stack
aws cloudformation create-stack --stack-name $FIRST_STACK --template-body file://1-bedrock-api-lambda-stack.yaml --capabilities CAPABILITY_IAM --region $AWS_REGION
```

#### To check status: 
```
aws cloudformation describe-stack-events --stack-name $FIRST_STACK --query "StackEvents[0]" --output table --region $AWS_REGION
```

- Wait for status to change to CREATE_COMPLETE

#### After stack is created, get the API Gateway Endpoint
```
API_ENDPOINT=$(aws cloudformation describe-stacks --stack-name $FIRST_STACK --query "Stacks[0].Outputs[?OutputKey=='ApiEndpoint'].OutputValue" --region $AWS_REGION --output text)

echo $API_ENDPOINT

```

#### Test Lamdba is able to invoke using API Gateway: 
```
curl -X POST $API_ENDPOINT -H 'Content-Type: application/json'   -d '{"input": "Tell me a joke"}'

```

Sample Output:
```
{"response": {"id": "msg_bdrk_013UrDiQjvw1YuGYr83Lph7m", "type": "message", "role": "assistant", "model": "claude-3-sonnet-20240229", "content": [{"type": "text", "text": "Here's a funny joke for you:\n\nWhy can't a bicycle stand up by itself? Because it's two-tired!"}], "stop_reason": "end_turn", "stop_sequence": null, "usage": {"input_tokens": 11, "output_tokens": 29}}}
```
- If face any issue, check the Lambda Log. 

### Step 3: CloudFormation to host a web page on S3 to intract with API Gateway and access using  CloudFront
#### Create stack

```
SECOND_STACK=cloudfront-s3-website
aws cloudformation create-stack --stack-name $SECOND_STACK --template-body file://2-cloudfront-s3-website.yaml --capabilities CAPABILITY_IAM --region $AWS_REGION
```

Check the status: 
```
aws cloudformation describe-stack-events --stack-name $SECOND_STACK --query "StackEvents[0]" --output table --region $AWS_REGION
```

- Wait for status to change to CREATE_COMPLETE


### Step 4: Upload the index.html to S3 bucket
#### Get the S3 Bucket to upload

```
S3_BUCKET=$(aws cloudformation describe-stacks --stack-name $SECOND_STACK --query "Stacks[0].Outputs[?OutputKey=='S3BucketName'].OutputValue" --region $AWS_REGION --output text)

echo $S3_BUCKET

```

#### Update line 39 in index.html with API Gateway URL: 
```
echo $API_ENDPOINT
```

Remove the Current value on line 39: 
```
            const API_GATEWAY_URL='https://nemxuql235.execute-api.us-west-2.amazonaws.com/prod/bedrock'
```

#### Copy the index.html to S3 bucket(from above output)
```
aws s3 cp index.html s3://$S3_BUCKET/
```

#### CloudFront to upload
```
CloudFrontDistributionId=$(aws cloudformation describe-stacks --stack-name $SECOND_STACK --query "Stacks[0].Outputs[?OutputKey=='CloudFrontDistributionId'].OutputValue" --region $AWS_REGION --output text)

echo $CloudFrontDistributionId

```

#### Invalidate CloudFront for the lasted html
```
aws cloudfront create-invalidation --distribution-id $CloudFrontDistributionId --paths "/*"
```

#### Open the web page to call bedrock: 

```
CloudFrontURL=$(aws cloudformation describe-stacks --stack-name $SECOND_STACK --query "Stacks[0].Outputs[?OutputKey=='CloudFrontURL'].OutputValue" --region $AWS_REGION --output text)

echo $CloudFrontURL
```


![steamlitapp](https://github.com/rahulsing/aws-bedrock-cloudfront-apigateway-lambda-demo/blob/main/CloudFrontHosted.PNG?raw=true)


### Clean up

#### Delete the second stack: 
```
aws s3 rm s3://$S3_BUCKET --recursive
aws cloudformation delete-stack --stack-name $SECOND_STACK --region $AWS_REGION
```

Check the status: 
```
aws cloudformation describe-stack-events --stack-name $SECOND_STACK --query "StackEvents[0]" --output table --region $AWS_REGION
```

#### Delete the first stack: 

```
aws cloudformation delete-stack --stack-name $FIRST_STACK --region $AWS_REGION
```

```
aws cloudformation describe-stack-events --stack-name $FIRST_STACK --query "StackEvents[0]" --output table --region $AWS_REGION
```
