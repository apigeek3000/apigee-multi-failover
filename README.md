# apigee-multi-failover

This repo outlines a resiliant architecture between Apigee and backend servers. If the a server returns a 503 error then Apigee will failover to the backup server, if the backup server also returns a 503 error then Apigee will call to PubSub to asynchronosly send the message, and finally Apigee will return a 202 message.

## Prereqs

There are a few things you'll need before starting:
- A GCP project where you are editor
- Your GCP project needs a deployed Apigee organization

## Create Service Account

Follow [the documentation to create a service account](https://cloud.google.com/iam/docs/service-accounts-create) named `failover-example` with the `pubsub.topics.publish`

## Deploy Cloud Run Functions

You will deploy two Cloud Run Functions, server-a and server-b. Follow [these instructions](https://cloud.google.com/run/docs/quickstarts/functions/deploy-functions-console#deploy_the_function) but with a few important configurations to note:
- Create two seperate functions, and name them server-a and server-b. They will have identical configurations and code (for now)
- Be sure to allow unauthenticated invocations (this is for demo purposes)
- To keep cost down, you can configure your function with the lowest configurations for Memory(128 MiB) and CPU (1)
- For runtime choose Node.js 22 (Ubuntu 22), make sure that the Function entry point is helloHttp, and use the following code for your function code in index.js
```
const functions = require('@google-cloud/functions-framework');

functions.http('helloHttp', (req, res) => {
  res.send(`Hello ${req.query.name || req.body.name || 'World'}!`);
});

// functions.http('helloHttp', (req, res) => {
//   res.sendStatus(503);
// });
```

Test our your Cloud Functions to make sure they work (the Function should show you a URL near the top)

## Deploy PubSub

Navigate to PubSub in the GCP and click the button to Create a Topic. Configure it as such:
- Name the function server-a-queue
- Uncheck "Add a default subscription"
- Check "Enable message retention"
- Leave everything else as default

After deploying the PubSub Topic, navigate back to the server-a Cloud Run Function. Enter the "Triggers" tab, select Add Trigger, and choose PubSub. In the side panel, keep all default settings besides topic. Choose the topic you just created.

Creating this trigger enables the PubSub Topic --> PubSub Subscription --> Trigger --> Cloud Run Function flow

## Deploy Apigee Proxy

Follow [the documentation to create two target servers](https://cloud.google.com/apigee/docs/api-platform/deploy/load-balancing-across-backend-servers#createtargetservers) for both server-a and server-b. Here are a few notes on the configuration:
- For the target server which points to the Cloud Run Function server-a, name it server-a. Likewise, do the same for server-b.
- For both target servers, configure them on port 443 and enable SSL (no further SSL config needed)

Deploy Apigee proxy (insert instructions here)

## Test Failover
