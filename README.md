# apigee-multi-failover

This repo outlines a resiliant architecture between Apigee and backend servers. If the a backend server returns a 503 error then Apigee will failover to the backup server, if the backup server also returns a 503 error then Apigee will call to PubSub to asynchronosly send the message to server-a to handle once it's back online.

## Prereqs

There are a few things you'll need before starting:
- A GCP project where you are editor
- Your GCP project needs a deployed Apigee organization

## Create Service Account

Follow [the documentation to create a service account](https://cloud.google.com/iam/docs/service-accounts-create) named `failover-example` with the `pubsub.topics.publish` permission.

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

Now let's deploy the Apigee proxy. Zip up the apiproxy folder and name the zip file `failover-example-v1.zip`. This will be our proxy bundle.

Navigate to Apigee in the GCP, go to API proxies, click the +CREATE button, under Proxy template choose Upload proxy bundle, upload our proxy bundle zip, name the proxy `failover-example-v1`, use the failover-example SA in the service account field, and deploy the proxy in your environment.

## Test Multi Failover Example

First, let's open a few tabs ahead of time. In two seperate tabs open up server-a and server-b and for each Cloud Run Function navigate to the Logs tab. Next, open a new tab enter your server-a-queue PubSub topic and go to the Metrics tab.

Now, let's see what happens when server-a responds with no issues. Make a POST request (consider using curl or Postman) to your newly deployed proxy at `https://your-apigee-domain.nip.io/v1/failover-example`. You'll get a 200 response and if you refresh the logs at server-a you should see a log for the 200 response returned.

Now, let's see what happens when server-a responds with a 503 error. In server-a go to the source code, choose to Edit source, comment out the function that is currently serving the function, and uncomment the function that sends a 503 error. Save and redeploy this function, then navigate back to Logs. When server-a is done deploying send another request to your failover-example API. You'll once again get a 200 response but this time the 200 response will come from server-b. Check the logs of server-a and server-b to confirm this. This is because Apigee failed over to server-b after server-a responded with a 503 error.

Now, let's see what happens when both server-a and server-b respond with 503 errors. Go to the source code of server-b now and make the same changes that were made to server-a. Save, redeploy, and navigate back to the logs. When server-b is done deploying send another request to your failover-example API. This time, you'll get back a 202 response, indicating that the request was accepted but that the processing hasn't been completed. What happened is that both server-a and server-b responded with 503 errors so Apigee instead published a message to PubSub for async submission once server-a is back online. Check the logs of server-a and server-b, you'll notice that PubSub is repeatedly making request to server-a with exponential backoff, this is also indicated in the PubSub metrics dashboards. 

Now, let's see what happens when server-a comes back online. Go to the source code of server-a, revert the changes made to fix the server, and redeploy. Eventually, you'll see the PubSub message come through in server-a's logs with a 200 successful response.

## Conclusion

Apigee thrives at fault handling and server failover (among many other things) but isn't the right tool for asynchronous message queues. For that we leverage the power of PubSub, and Apigee's PublishMessage policy designed explicitely for invoking PubSub. This is just one approach to fault handling with Apigee, however, and for a more nuanced discussion feel free to read through [this Apigee Community post](https://www.googlecloudcommunity.com/gc/Apigee/Exception-Handling-Retrying-and-maintaining-state-capabilities/m-p/34137).

## Optional: PubSub Dead-letter topics

It may be a good idea to supplant this with a [PubSub dead-letter topic](https://cloud.google.com/pubsub/docs/handling-failures#dead_letter_topic). This allows you to define a strategy for storing & rerouting messages that cannot send to a target backend despite exponential backoff over time.

## Extra: Application Integration

Application Integration is an Integration-Platform-as-a-Service (IPAAS) that thrives at handling long-running asynchonous requests. See an example of how to implement an architecture similar to this Apigee + PubSub solution in the documentation [here](https://cloud.google.com/application-integration/docs/error-handling#example)