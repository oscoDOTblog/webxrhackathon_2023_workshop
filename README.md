# Workshop Part One

Welcome to the step-by-step guide to building your game backend using AWS services like API Gateway, Lambda, and DynamoDB. Follow the steps below to set up the backend infrastructure efficiently.

![webxr_test](images/webxr_test.gif)

## Prerequisites

Ensure that you have the AWS CDK installed and configured on your machine. If not, refer to the [main README](https://github.com/felixtrz/webxrhackathon_2023_workshop/blob/main/readme.md) for instructions.

Clone the workshop to your local machine

```
git clone git@github.com:felixtrz/webxrhackathon_2023_workshop.git
```

Check out the `workshop_one` branch

```
git checkout workshop_one
```

Install [Node.js](https://nodejs.org/en/download) (skip if already installed; recommend Node version 14.15.0+)

Open the code locally using your preferred IDE such as Visual Studio Code

## Step 1: Clean up CDK resources and web builds

Ensure a clean slate by removing any previously compiled CDK resources. Go to the `infra` folder from the project root, and delete the cloud assembly folder (`cdk.out`, if it exists):

```
rm -rf cdk.out
```

Similarly, go to the `web` folder from the project root, and delete the `dist` folder (if it exists):

```
rm -rf dist
```

## Step 2: Install dependencies

Go to the `infra` folder, and run:

```
npm install
```

Go to the `web` folder, and run:

```
npm install
```

## Step 3: Set up Lambda functions

### 3.1 Create Lambda functions

Go to the `infra/lib` folder, open `main.ts`, and uncomment the following lines to declare two Lambda functions:

```javascript
const getHighScoreLambda = new LambdaStack(
  scope,
  "getPlayerInfoLambda",
  cdk.aws_lambda.Runtime.NODEJS_18_X,
  "../lambdaScripts/getPlayerInfo",
  "handler",
  cdk.Duration.minutes(5),
  512,
  512,
  highScoreEnvs
);

const putHighScoreLambda = new LambdaStack(
  scope,
  "putPlayerRecordLambda",
  cdk.aws_lambda.Runtime.NODEJS_18_X,
  "../lambdaScripts/putPlayerRecord",
  "handler",
  cdk.Duration.minutes(5),
  512,
  512,
  highScoreEnvs
);
```

These lines create two Lambda functions:

- getPlayerInfoLambda - retrieves player information.
- putPlayerRecordLambda - updates player information by setting player ID and score.

The actual Lambda function code is located within `../lambdaScripts/getPlayerInfo` and `../lambdaScripts/putPlayerRecord`respectively.

### 3.2 Implement Lambda functions code

Go to the `infra/lambdaScripts` directory.

Open `putPlayerRecord/putPlayerRecord.js`, and uncomment the following code block:

```javascript
export const handler = async (event, context) => {
  const body = JSON.parse(event.body);
  try {
    const params = {
      TableName: leaderboardTable,
      Item: {
        playerId: { S: body.playerId },
        score: { N: `${body.score}` },
        status: { N: "1" },
      },
    };
    await dbClient.send(new PutItemCommand(params));
  } catch (err) {
    return JsonResponse(500, "Error storing high score.");
  }

  return JsonResponse(200, "High score stored.");
};
```

This code enables the Lambda function to create or update player information in DynamoDB.

Open `getPlayerInfo/getPlayerInfo.js`, and uncomment the following code block:

```javascript
export const handler = async (event, context) => {
  try {
    const playerId = event.pathParameters.playerId.toLowerCase();

    const recordScore = await getPlayerRecordScore(playerId);

    const worldData = await getWorldRecordAndRanking(playerId);

    const response = {
      recordScore: recordScore,
      worldRecord: worldData.worldRecord,
      ranking: worldData.ranking,
    };

    return JsonResponse(200, response);
  } catch (err) {
    console.log(err);
    return JsonResponse(500, "Error getting player info.");
  }
};
```

This code enables the Lambda function to retrieve player score information and ranking from DynamoDB.

## Step 4: Implement API using API Gateway

Return to the `infra/lib` folder, open `main.ts`, and uncomment the API Gateway code block:

```javascript
const apiGateway = new restGatewayNestedStack(
  scope,
  "gateway",
  "Main Stack Gateway",
  "dev"
).gateway;
apiGateway.AddMethodIntegration(
  putHighScoreLambda.MethodIntegration(),
  "leaderboard",
  "POST"
);
apiGateway.AddMethodIntegration(
  getHighScoreLambda.MethodIntegration(),
  "leaderboard/{playerId}",
  "GET"
);
```

These lines create two API endpoints:

- One for creating/updating player information.
- One for retrieving player information.

Each API is integrated with the respective Lambda functions crafted earlier.

All the steps above are staging the changes in the CDK project. The next step will deploy the services using the CDK CLI.

## Step 5: Build the app and deploy resources

Go to the `web` folder from the project root, and run this command to build the app:

```
npm run build
```

Go to the `infra` folder from the project root, and run this command to deploy the CDK:

```
cdk deploy --outputs-file './config.json'
```

Wait for the deployment to finish and take note of all of the outputs. You will refer to them later on.

If you're new to CDK, `cdk deploy` is essentially synthesizing a CloudFormation template and launching the template with CloudFormation. CDK libraries enable you to express your AWS resources using common programming languages like TypeScript, Python, Java, and more. Not only can you apply programming logic like loops and conditions, CDK constructs reduce the amount of code you need to write compared to directly authoring CloudFormation templates and implements sane defaults.

This is what a successful deployment output will look like -

![Deployment result](images/deployment_result.png)

## Step 6: Test the APIs

### 6.1 Store a high score

Utilize cURL to test the POST API endpoint. Ensure to replace the endpoint URL.

```
curl -X POST <REPLACE_WITH_WorkshopOne.gatewayGEndpoint_FROM_CDK_OUTPUT>/leaderboard  -H "Content-Type: application/json" -d '{"playerId": "PlayerId123", "score": "123"}'
```

Upon successful execution, you should observe a success message: "High Score stored." Validate the data insertion by navigating to the DynamoDB table via the AWS management console.

![dynamoDB](images/dynamoDB.png)

### 6.2 Retrieve Player Information

Test the GET API endpoint to retrieve player information. Ensure to replace the endpoint URL.

```
curl -X GET <REPLACE_WITH_WorkshopOne.gatewayGEndpoint_FROM_CDK_OUTPUT>/leaderboard/PlayerId123 -H "Content-Type: application/json"
```

A successful query will return player information in the following format:

```
{"recordScore":123,"worldRecord":123,"ranking":1}
```

🎉 Congratulations! 🎉

You've learned how to build backend APIs using AWS services and will integrate them into your game code in the subsequent stages.

# Part Two

This guide walks through integrating the backend API with your game. Then you will run, build, and host your game on CloudFront and S3.

## Step 1. Integrate backend API

Ensure you have your backend API URL from CDK outputs.
Navigate to the `web/src` folder, open `amplifyconfigure.js`, and replace the placeholder with your API Gateway URL from CDK outputs.

```
url: '<REPLACE_WITH_WorkshopOne.gatewayGEndpoint_FROM_CDK_OUTPUT>'
```

## Step 2. Testing locally

Navigate to the `web/` folder and run:

```
npm run serve
```

This will start your game locally with the integrated backend in the cloud. Please use Chrome for testing.

First, install the [WebXR API Emulator plugin](https://chrome.google.com/webstore/detail/webxr-api-emulator/mjddjgeghkdijejnciaefnkjmkafnnje) (skip if the extension is already installed)

Open [localhost:8081](https://localhost:8081/) in a browser to load the game.

Right-click anywhere in the browser window, and select "Inspect" to open Chrome developer tools.

Open the `WebXR` panel from the top of the developer tools frame.

![WebXR panel from DevTools](images/panel.png)

Click the "Enter VR" button to start VR mode and spawn the player. Use the handles of the simulated XR device to interact. Flapping up and down three times will initiate the game. In the `Network` tab of the Chrome developer tools, you can verify if the API call was successful.

![WebXR plugin](images/plugin-demo.gif)

# Part Three

This guide walks through preparing a production build and pushing the application code to S3.

## Step 1. Preparing for production

To a production build, run this under web folder

```
npm run build
```

This build is ready for uploading to the specified S3 bucket.

## Step 2. Hosting the game

You can use either the [AWS management console](#using-the-aws-management-console) or [AWS CLI](#using-the-aws-cli) to push your application files to S3, choose your adventure.

### Using the AWS management console

Refer to the screenshots below for guidance on how to upload to the S3 bucket.

Open the AWS management console, and navigate to the [Amazon S3 console](https://s3.console.aws.amazon.com/s3/home).

Select your provisioned S3 bucket (bucket name can be found in CDK outputs), and click "Upload"

![S3 Bucket](images/s3_front.png)

Upload every file in the `web/dist` folder (except for /assets, because these were uploaded through CDK deloy) into your S3 bucket.

![S3 Bucket Upload](images/s3_upload.png)

### Using the AWS CLI

Navigate to the `web/dist` folder in your terminal.

Run the following AWS CLI command (swapping in your S3 bucket) to upload your application code:

```
aws s3 cp . s3://<REPLACE_WITH_S3_BUCKETNAME_FROM_CDK_OUTPUT> --recursive
```

The value of <REPLACE_WITH_S3_BUCKETNAME_FROM_CDK_OUTPUT> will be right after the "bucket" in CDK output. As an example, "workshopone-storagebucket19db2ff8-10f6mt30q5nfu" is the bucket name in the sample CDK deployment, you should replace <REPLACE_WITH_S3_BUCKETNAME_FROM_CDK_OUTPUT> with "workshopone-storagebucket19db2ff8-10f6mt30q5nfu".

```
AWSS3":{"bucket":"workshopone-storagebucket19db2ff8-10f6mt30q5nfu"}
```

After uploading, open the CloudFront URL (can be found in CDK outputs from the previous CDK deploy step) in a browser to see your game in action.

As a refresher, here's sample CDK output:

```
WorkshopOne.Params = {"Storage":{"AWSS3":{"bucket":"workshopone-storagebucket19db2ff8-10f6mt30q5nfu"}},"CloudFront":{"domainName":"d34uvascesby51.cloudfront.net"},"API":{"endpoints":[{"name":"gateway_G","endpoint":"https://14avt94r5j.execute-api.us-east-1.amazonaws.com/dev/","region":"us-east-1"}]}}
WorkshopOne.gatewayGEndpoint5D1E33B9 = https://14avt94r5j.execute-api.us-east-1.amazonaws.com/dev/
```

The CloudFront URL in the above output would be: **d34uvascesby51.cloudfront.net**

Remember, use the "Network" tab in developer tools to help verify the correct API Gateway endpoint and detect any potential CORS issues.

# Jump to workshop two:

You're all set to progress to workshop two, click here -> [workshop two](https://github.com/felixtrz/webxrhackathon_2023_workshop/tree/workshop_two) or select "workshop_two" from this repository's branches to access the README.
