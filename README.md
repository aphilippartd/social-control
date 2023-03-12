# Social Hater Demo

## Prerequisites

In order to proceed with the demo, you should have the following:
- [AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html) installed
- [AWS SAM](https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/install-sam-cli.html) installed
- [Configure your AWS credentials](https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-configure.html)

## What we are building

This repository contains a demo using AWS Application Composer and serverless technologies. We will be building an API allowing you to post messages and get notified whenever a message is identified as negative.

## Architecture

![plot](./images/architecture.png)

## Reproduction steps

### Define our infrastructure with AWS Application Composer

In this demo we will start building an API. In order to do so, we will be leveraging [AWS Application Composer](https://aws.amazon.com/application-composer/).

1. Create a directory in which you wish to create your project on your local computer
2. Head to the [Application Composer console](https://console.aws.amazon.com/composer/home)
3. Click on **create project**
4. Select **New Blank Project**
5. Select **Connected**
6. Select the your local directory
7. Click **Create**
8. Drag and drop an API Gateway resource into the canvas
9. Click on the resource, then details and call it **SocialHaterAPI**
10. Change the method from GET to POST
11. Click save 
    ![plot](./images/infra/1.png)
12. Drag and drop a Lambda resource into the canvas
13. Change the logical ID to **SocialHaterQueuer**
14. Change the source path to **src/Queuer**
15. Click save
    ![plot](./images/infra/2.png)
16. Link the POST method of **SocialHaterAPI** with **SocialHaterQueuer**
17. Drag and drop an SQS Queue resource in the canvas
18. Change the logical ID to **SocialHaterQueue**
19. Click save
  ![plot](./images/infra/3.png)
20. Link **SocialHaterQueuer** with **SocialHaterQueue**
21. Drag and drop a Lambda resource into the canvas
22. Change the logical ID to **SocialHaterHandler**
23. Change the source path to **src/Handler**
24. Click save
  ![plot](./images/infra/4.png)
25. Link Subscription of the **SocialHaterQueue** with **SocialHaterHandler**
  [plot](./images/infra/5.png)
26. Click on the **SocialHaterHandler** resource details and scroll down to **Permissions**
27. Paste the following code
    ```yaml
    - Statement:
        - Effect: Deny
          Action:
            - sns:Publish
          Resource: arn:aws:sns:*:*:*
        - Effect: Allow
          Action:
            - sns:Publish
          Resource: '*'
    - Statement:
        - Effect: Allow
          Action:
            - comprehend:DetectSentiment
          Resource: '*'
    ```
    These policies will:
      1. Allow the lambda function to send SMS through the Amazon SNS service (but not to acces any topic)
      2. Allow the lambda to perform the DetectSentiment command on Amazon Comprehend
28. Click save
    ![plot](./images/infra/6.png)


### Define lambda execution logic

Now we want to code the execution logic of our lambda functions in our local computer.

1. Head to the directory you connected with Application Composer.
2. Open your `template.yaml` file. This file contains all your SAM configuration and will be used to deploy resources on AWS.
3. Head to the end of the `template.yml` document and paste the following code:
    ```yaml
    Outputs:
      MvpStoriesApi:
        Description: "API Gateway endpoint URL for Prod stage"
        Value: !Sub "https://${SocialHaterAPI}.execute-api.${AWS::Region}.amazonaws.com/Prod/"
    ```
    This change will make sure that everytime you deploy your application with SAM, the API Gateway endpoint will be displayed.
4. Remove the files and folders in the `src` directory
5. Create two directories in the `src` directory:
   - `src/Queuer`
   - `src/Handler`
6. In each of those directories create an `index.mjs` file
7. Let's head to the `src/Queuer/index.mjs` file and paste the following code:
    ```js
    import { SQSClient, SendMessageCommand } from "@aws-sdk/client-sqs"

    const sqsClient = new SQSClient()

    export const handler = async (event) => {
      const sqsInput = { QueueUrl: process.env.QUEUE_URL, MessageBody: event.body }
      const sqsCommand = new SendMessageCommand(sqsInput)
      await sqsClient.send(sqsCommand)
      return {}
    }
    ```
    This codes adds a message in the SQS queue. The message is the body of the API call made on the API gateway endpoint.
8. Let's head to the `src/Handler/index.mjs` file and paste the following code:
    ```js
    import { SNSClient, PublishCommand } from "@aws-sdk/client-sns"
    import { ComprehendClient, DetectSentimentCommand } from "@aws-sdk/client-comprehend"

    export const handler = async (event) => {
      const message = JSON.parse(event.Records[0].body)
      
      const comprehendInput = { Text: message.content, LanguageCode: "en" }
      const comprehendClient = new ComprehendClient()
      const comprehendCommand = new DetectSentimentCommand(comprehendInput)
      const comprehendResponse = await comprehendClient.send(comprehendCommand)

      console.log(`Sentiment is ${comprehendResponse.Sentiment}`)
      if (comprehendResponse.Sentiment === "NEGATIVE") {
        const snsInput = { Message: `ALERT: Hater message received from ${message.sender || "UNKNOWN HATER"} (ID: ${message.id})`, PhoneNumber: "+971505007832" }
        const snsClient = new SNSClient()
        const snsCommand = new PublishCommand(snsInput)
        await snsClient.send(snsCommand)
        console.log("SMS Sent")
      }

      return {}
    }
    ```
    This code extracts the message from SQS. It then makes a call to Amazon Comprehend to assess if the message is negative. If it is negative it sends a message to the phone number hard coded in this code snippet.
    > NOTE: For testing purposes, you need to add your phone number to the SNS service before being able to send SMS messages to your phone number.

  ### Deploying our API

  Now that we have all the configuration of our application ready, let's deploy it.

  1. At the root of your project run `sam build` this will build your deployment files into a directory called `.aws-sam`
  2. Run `sam deploy --guided`
   This command will ask for some information regarding your deployment, you can fill the values in as follows:
   ![plot](./images/deploy/1.png)
   > Note: It will warn you that **SocialHaterQueuer** has no Authorization defined. Explicetely allow it for the sake of this demo.
  3. At the end of your deployment results, you should be able to find the outputs that should look like this:
  ![plot](./images/deploy/2.png)
  Copy the API endpoint and test your api with the following command
  ```sh
    curl --location 'API_ENDPOINT' \
    --header 'Content-Type: application/json' \
    --data '{
        "content": "Not happy at all with the service",
        "sender": "Angry sender",
        "id": "Angry ID"
    }'
  ```
  Copy the API endpoint and test your api with the following command
  ```sh
    curl --location 'API_ENDPOINT' \
    --header 'Content-Type: application/json' \
    --data '{
        "content": "Very happy with the service",
        "sender": "Happy sender",
        "id": "Happy ID"
    }'
  ```

  