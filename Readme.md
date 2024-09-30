## Step 1: Clone GitHub Role: 
```
git clone 
```

## Step 2: CloudFormation to create API Gateway and Lamdba Function Invoking Bedrock. 

#### Create stack
```
AWS_REGION="us-west-2"
FIRST_STACK=bedrock-api-lambda-stack
aws cloudformation create-stack --stack-name $FIRST_STACK --template-body file://1-bedrock-api-lambda-stack.yaml --capabilities CAPABILITY_IAM --region $AWS_REGION

```
#### After stack is created, get the API Gateway Endpoint
```
API_ENDPOINT=$(aws cloudformation describe-stacks --stack-name $FIRST_STACK --query "Stacks[0].Outputs[?OutputKey=='ApiEndpoint'].OutputValue" --region $AWS_REGION --output text)

echo $API_ENDPOINT

```

#### Test Lamdba is able to invoke : 
```
curl -X POST 'https://jk8hu8ezg4.execute-api.us-west-2.amazonaws.com/prod/bedrock'   -H 'Content-Type: application/json'   -d '{"input": "Tell me a joke"}'

```

## Step 3: CloudFormation to host a web page on S3 to intract with API Gateway and access using  CloudFront
#### Create stack

```
SECOND_STACK=cloudfront-s3-website
aws cloudformation create-stack --stack-name $SECOND_STACK --template-body file://2-cloudfront-s3-website.yaml --capabilities CAPABILITY_IAM --region $AWS_REGION
```

## Step 4: Upload the index.html to S3 bucket
#### S3 Bucket to upload

```
S3=$(aws cloudformation describe-stacks --stack-name $SECOND_STACK --query "Stacks[0].Outputs[?OutputKey=='WebsiteURL'].OutputValue" --region $AWS_REGION --output text)

echo $S3

```
#### CloudFront to upload
```
CloudFrontURL=$(aws cloudformation describe-stacks --stack-name $SECOND_STACK --query "Stacks[0].Outputs[?OutputKey=='CloudFrontURL'].OutputValue" --region $AWS_REGION --output text)

echo $CloudFrontURL

```

```
aws cloudfront create-invalidation --distribution-id E1M1IBYR78IHL2 --paths "/*"
```