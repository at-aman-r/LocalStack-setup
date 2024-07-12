# LOCALSTACK SETUP GUIDE
---
## Setting Up `LocalStack` a local simulation of AWS services, suitable for development and testing purposes.

---

### Prerequesits: 
1. [Node.js]([OpenAI](https://www.openai.com))
2. [Python]([https://www.python.org/downloads/])
3. [PIP]([https://pip.pypa.io/en/stable/installation/])
4. [Docker]([https://www.docker.com/products/docker-desktop/])

###### Note: the enviroment variables `path` must be set properly eg.
```env
C:\Program Files\nodejs\
C:\Python\Python 311
C:\Python\Python 311\Scripts
C:\Program Files\Docker\Docker\resources\bin
C:\Program Files\Amazon\AWSCLIV2\

(Your Installation location might be different)
```

---

### STEP 1:
#### Installation of Localstack:
> 1. Install the **Localstack CLI**
   
   [Download Link]({https://app.localstack.cloud/getting-started})

   ``No need to configure your personal auth token. it is for paid version``

> 2. Install the **AWS CLI**

   ##### Using PIP

   `pip install awscli` 

> 3. Install **Localstack AWS CLI** (`awslocal`)
#### Using PIP

    ``pip install awscli-local[ver1]``

---

### Step 2:
> **Configration Of Localstack AWS CLI** 



   ##### 1. Open CMD and execute 

   > `awslocal configure`


    AWS Access Key ID [ ]:  test
    AWS Secret Access Key [ ]: test
    Default region name [ ]: us-east-1
    Default output format [ ]: json

---

### Step 3: 
>   **Starting the Localstack in docker**

    1. Open Docker 
    2. Open CMD and execute below command:
`localstack start  `



   `⁜ the localstack will start runiing in a docker container`

   `Now the setup is complete we can move to testing the setup`
   
---

### Step 4:    
> **Testing the installation**


##### [Detailed Doc LInk..]([https://docs.localstack.cloud/user-guide/aws/apigateway/])


#### 1. Create a new file named lambda.js with the following contents:

```javascript
'use strict'

const apiHandler = (payload, context, callback) => {
    callback(null, {
        statusCode: 200,
        body: JSON.stringify({
            message: 'Hello from Lambda'
        }),
    }); 
}
    
module.exports = {
    apiHandler,
}

```


#### 2. Zip the lambda.js filee
 ##### Using CMD:
`zip function.zip lambda.js`


#### 3. Upload it to LocalStack using the `awslocal CLI`. Run the following command:

```cmd
awslocal lambda create-function 
  --function-name apigw-lambda 
  --runtime nodejs16.x 
  --handler lambda.apiHandler 
  --memory-size 128 
  --zip-file fileb://function.zip 
  --role arn:aws:iam::111111111111:role/apigw
  ```

###### Note: Here  
```cmd
awslocal lambda create-function 
  --function-name apigw- <Lambda Function Name>
  --runtime <Runtime with version without space eg python8.11, nodejs20.x> 
  --handler lambda.apiHandler 
  --memory-size 128 
  --zip-file fileb://<Name of the zip file>.zip 
  --role arn:aws:iam::111111111111:role/apigw
  ``` 


####  4. Create a REST API 

###### We will use the API Gateway’s CreateRestApi API to create a new REST API. Here’s an example command:

`awslocal apigateway create-rest-api --name 'API Gateway Lambda integration'`

##### This creates a new REST API named API Gateway Lambda integration. The above command returns the following response:

```json 
{
    "id": "cor3o5oeci",
    "name": "API Gateway Lambda integration",
    "createdDate": "2023-04-27T16:08:46+05:30",
    "apiKeySource": "HEADER",
    "endpointConfiguration": {
        "types": [
            "EDGE"
        ]
    },
    "disableExecuteApiEndpoint": false
} 
```

Note the REST API ID returned in the response. You’ll need this ID for the next step.

#### 5. Fetch the Resources
###### Use the REST API ID generated in the previous step to fetch the resources for the API, using the GetResources API:

`awslocal apigateway get-resources --rest-api-id <REST_API_ID>`


The above command returns the following response:
``` json
{
    "items": [
        {
            "id": "u53af9hm83",
            "path": "/"
        }
    ]
}
```


Note the ID of the root resource returned in the response. You’ll need this ID for the next step.

#### 6.Create a resource

Create a new resource for the API using the CreateResource API. Use the ID of the resource returned in the previous step as the parent ID:
```cmd 
awslocal apigateway create-resource 
  --rest-api-id <REST_API_ID> 
  --parent-id <PARENT_ID> 
  --path-part "{somethingId}"
```

The above command returns the following response:
``` json
{
    "id": "zzcvcf56ar",
    "parentId": "u53af9hm83",
    "pathPart": "{somethingId}",
    "path": "/{somethingId}"
}
```

Note the ID of the root resource returned in the response. You’ll need this Resource ID for the next step.

 #### 7. Add a method and integration
###### Add a GET method to the resource using the PutMethod API. Use the ID of the resource returned in the previous step as the Resource ID:
```cmd
awslocal apigateway put-method 
  --rest-api-id <REST_API_ID> 
  --resource-id <RESOURCE_ID> 
  --http-method GET 
  --request-parameters "method.request.path.somethingId=true" 
  --authorization-type "NONE"
```

The above command returns the following response:
```json
{
    "httpMethod": "GET",
    "authorizationType": "NONE",
    "apiKeyRequired": false,
    "requestParameters": {
        "method.request.path.somethingId": true
    }
}
```


Now, create a new integration for the method using the PutIntegration API.

``` cmd
awslocal apigateway put-integration 
  --rest-api-id <REST_API_ID> 
  --resource-id <RESOURCE_ID> 
  --http-method GET 
  --type AWS_PROXY 
  --integration-http-method POST 
  --uri arn:aws:apigateway:us-east-1:lambda:path/2015-03-31/functions/arn:aws:lambda:us-east-1:000000000000:function:apigw-lambda/invocations 
  --passthrough-behavior WHEN_NO_MATCH

```

The above command integrates the GET method with the Lambda function created in the first step. We can now proceed with the deployment before invoking the API.

#### 8.Create a deployment
Create a new deployment for the API using the CreateDeployment API:
```cmd
awslocal apigateway create-deployment 
  --rest-api-id <REST_API_ID> 
  --stage-name test
```

> Your API is now ready to be invoked. You can use cURL or any HTTP REST client to invoke the API endpoint:

`curl -X GET http://localhost:4566/restapis/<REST_API_ID>/test/_user_request_/test`

###### Now `http://localhost:4566/restapis/<REST_API_ID>/test/_user_request_/test` this is your API and output here is:

> `{"message":"Hello from Lambda"}   `


#### Helful Docs Refs:
1. [Lambda funtion and api gateway in localstack]([https://docs.localstack.cloud/user-guide/aws/apigateway/])

2. [Cognito on LocalStack]([https://docs.localstack.cloud/user-guide/aws/cognito/])

3 . [ DynamoDB on LocalStack]([https://docs.localstack.cloud/user-guide/aws/dynamodb/])

4 . [Simple Notification Service (SNS) on LocalStack]([https://docs.localstack.cloud/user-guide/aws/sns/])

5 . [All Local AWS Services ]([https://docs.localstack.cloud/user-guide/aws/feature-coverage/])











