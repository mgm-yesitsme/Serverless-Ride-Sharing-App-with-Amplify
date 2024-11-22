*** Overview:

When I started exploring AWS, I wanted a project that would help me connect the dots between its services while giving me practical, resume-worthy experience. That’s when I decided to build a Ride-Sharing App for Unicorns, inspired by the AWS Wild Rydes sample project!

This project combines seven AWS services—Amplify, Cognito, Lambda, IAM, API Gateway, DynamoDB, and GitHub—into a serverless architecture. It was both a rewarding and challenging journey, but it gave me invaluable hands-on experience with AWS.
** Big Thanks to @tinytutorials for the dependencies and some tips..

*** Prerequisites:

I made sure to have:

An AWS account with admin permissions.
A GitHub account for source control.
Basic knowledge of AWS services like Lambda, DynamoDB, and Amplify.

*** Hosting the Application with AWS Amplify
Setting Up Amplify:

I logged into the AWS Console, navigated to Amplify, and connected it to my GitHub repository.
Amplify automatically detected my project and deployed it, giving me a URL for my application.
Testing CI/CD:
I made small updates to the codebase (e.g., changing UI text) and pushed them to GitHub. Amplify picked up these changes and rebuilt the app automatically, showcasing its CI/CD capabilities.

* Challenge: The initial build failed because of an incorrect file structure.
Solution: After inspecting the build logs in Amplify, I fixed the directory structure and adjusted the buildspec.yml file. This taught me the importance of aligning code with expected deployment configurations.

*** Adding User Authentication with Amazon Cognito
Creating a Cognito User Pool:
I configured a new user pool in Cognito to handle user registration and authentication.

Integrating Cognito with Amplify:
Updated the config.js file in the app with the Cognito user pool details.

Testing Authentication:

Signed up a test user and verified the registration flow.
Logged in with the test user to ensure authentication worked.
Challenge: Debugging user login issues.
Solution: I realized I had forgotten to enable the user pool in the Amplify app settings. After enabling it, login worked perfectly.

*** Backend Logic with AWS Lambda and DynamoDB
Setting Up DynamoDB:

Created a table named Rides with RideId as the primary key to store ride details.
Creating the Lambda Function:

I wrote a Lambda function in Python that assigns a unicorn to a user and stores the ride details in DynamoDB.
Attached an IAM role to the function with PutItem permissions for the Rides table.
Testing Lambda:

Deployed the function and tested it with a sample event.
Verified the data was saved correctly in DynamoDB.
Challenge: The Lambda function initially failed to write to DynamoDB.
Solution: I realized the IAM role didn’t have the required dynamodb:PutItem permission. After updating the policy, the function worked as expected.

*** Exposing the Backend with API Gateway
Creating a REST API:

I set up an API Gateway to expose the Lambda function as a REST API.
Adding Cognito Authorization:

Configured an authorizer in API Gateway to validate tokens from Cognito.
Configuring API Resources:

Added a POST /rides endpoint for booking rides.
Enabled CORS to allow requests from the front-end.
Testing the API:

Called the API endpoint using Postman to ensure the integration worked.
Verified ride details were saved in DynamoDB.
Challenge: API Gateway returned a 403 error during testing.
Solution: After investigating, I found that my Lambda function’s execution role was missing permissions to invoke the API. Once I added the necessary permissions, it worked perfectly.

*** Testing the Application
With all components in place, I tested the app end-to-end:

Registered a new user via Cognito and logged in.
Booked a ride, triggering the Lambda function and storing data in DynamoDB.
Checked the DynamoDB table to confirm the ride details were saved.
Seeing everything come together was incredibly satisfying!


*** Challenges I faced:
Understanding Service Integrations:
At first, it was overwhelming to figure out how all the AWS services worked together.

Solution: I broke the project into smaller parts, focusing on one service at a time, which made the process more manageable.
CI/CD Pipeline Failures in Amplify:
My initial build in Amplify failed due to incorrect file structures.

Solution: Reading the build logs in Amplify helped me identify and fix the issues.
Lambda Role Misconfigurations:
Lambda couldn't write to DynamoDB because of missing permissions.

Solution: I updated the IAM role to include dynamodb:PutItem permissions.
API Gateway Authorization Errors:
I faced multiple authorization errors while testing API Gateway.

Solution: After debugging, I ensured the API Gateway authorizer was correctly configured to use Cognito tokens.


** Lessons Learned **
This project taught me:

- The importance of IAM roles and permissions when working with AWS services.
- How to debug and resolve issues using AWS logs (CloudWatch, Amplify build logs, etc.).
- How to design and deploy a fully serverless application using AWS best practices.
- 
Future Enhancements
To make the app even better, I plan to:

- Add real-time ride updates using AWS AppSync.
- Implement push notifications for ride confirmations using Amazon SNS.
- Introduce analytics using AWS CloudWatch and AWS Pinpoint.
- This project was a challenging but rewarding journey, and I’m excited to apply these skills to future projects. Feel free to explore my repository and share your feedback! 🎉

**** Let’s Connect on LinkedIn : linkedin.com/in/godfrey-mesembe-5b33b42ba ****




## The Application Code
The application code is here in this repository.

## The Lambda Function Code
Here is the code for the Lambda function, originally taken from the [AWS workshop](https://aws.amazon.com/getting-started/hands-on/build-serverless-web-app-lambda-apigateway-s3-dynamodb-cognito/module-3/ ), and updated for Node 20.x:

```node
import { randomBytes } from 'crypto';
import { DynamoDBClient } from '@aws-sdk/client-dynamodb';
import { DynamoDBDocumentClient, PutCommand } from '@aws-sdk/lib-dynamodb';

const client = new DynamoDBClient({});
const ddb = DynamoDBDocumentClient.from(client);

const fleet = [
    { Name: 'Willy', Color: 'White', Gender: 'Male' },
    { Name: 'Ryhana', Color: 'Black', Gender: 'Female' },
    { Name: 'Carlo', Color: 'Green', Gender: 'Male' },
];

export const handler = async (event, context) => {
    if (!event.requestContext.authorizer) {
        return errorResponse('Authorization not configured', context.awsRequestId);
    }

    const rideId = toUrlString(randomBytes(16));
    console.log('Received event (', rideId, '): ', event);

    const username = event.requestContext.authorizer.claims['cognito:username'];
    const requestBody = JSON.parse(event.body);
    const pickupLocation = requestBody.PickupLocation;

    const unicorn = findUnicorn(pickupLocation);

    try {
        await recordRide(rideId, username, unicorn);
        return {
            statusCode: 201,
            body: JSON.stringify({
                RideId: rideId,
                Unicorn: unicorn,
                Eta: '30 seconds',
                Rider: username,
            }),
            headers: {
                'Access-Control-Allow-Origin': '*',
            },
        };
    } catch (err) {
        console.error(err);
        return errorResponse(err.message, context.awsRequestId);
    }
};

function findUnicorn(pickupLocation) {
    console.log('Finding unicorn for ', pickupLocation.Latitude, ', ', pickupLocation.Longitude);
    return fleet[Math.floor(Math.random() * fleet.length)];
}

async function recordRide(rideId, username, unicorn) {
    const params = {
        TableName: 'Rides',
        Item: {
            RideId: rideId,
            User: username,
            Unicorn: unicorn,
            RequestTime: new Date().toISOString(),
        },
    };
    await ddb.send(new PutCommand(params));
}

function toUrlString(buffer) {
    return buffer.toString('base64')
        .replace(/\+/g, '-')
        .replace(/\//g, '_')
        .replace(/=/g, '');
}

function errorResponse(errorMessage, awsRequestId) {
    return {
        statusCode: 500,
        body: JSON.stringify({
            Error: errorMessage,
            Reference: awsRequestId,
        }),
        headers: {
            'Access-Control-Allow-Origin': '*',
        },
    };
}
```

## The Lambda Function Test Function
Here is the code used to test the Lambda function:

```json
{
    "path": "/ride",
    "httpMethod": "POST",
    "headers": {
        "Accept": "*/*",
        "Authorization": "eyJraWQiOiJLTzRVMWZs",
        "content-type": "application/json; charset=UTF-8"
    },
    "queryStringParameters": null,
    "pathParameters": null,
    "requestContext": {
        "authorizer": {
            "claims": {
                "cognito:username": "the_username"
            }
        }
    },
    "body": "{\"PickupLocation\":{\"Latitude\":47.6174755835663,\"Longitude\":-122.28837066650185}}"
}
```

