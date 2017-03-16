# OpenWhisk building block - Cloudant Trigger
Create Cloudant data processing apps with Apache OpenWhisk on IBM Bluemix. This tutorial takes less than 5 minutes to complete. After this, move on to more complex serverless applications such as those tagged [_openwhisk-hands-on-demo_](https://github.com/search?q=topic%3Aopenwhisk-hands-on-demo+org%3AIBM&type=Repositories).

If you're not familiar with the OpenWhisk programming model [try the action, trigger, and rule sample first](https://github.com/IBM/openwhisk-action-trigger-rule). [You'll need a Bluemix account and the latest OpenWhisk command line tool](docs/OPENWHISK.md).

This example shows how to create an action that can be integrated with the built in Cloudant changes trigger and read action to execute logic when new data is added.

1. [Configure Cloudant](#1-configure-cloudant)
2. [Create OpenWhisk actions](#2-create-openwhisk-actions)
3. [Clean up](#3-clean-up)

# 1. Configure Cloudant
Log into Bluemix, create a [Cloudant database instance](https://console.ng.bluemix.net/catalog/services/cloudant-nosql-db/), and name it `openwhisk-cloudant`. Launch the Cloudant web console and create a database named `cats`. Extract the username and password from the "Service Credentials" tab in Bluemix.

In this demo, we will make use of the Cloudant package, which contains a set of OpenWhisk actions and feeds that integrate with a Cloudant database. Use the OpenWhisk CLI to bind the Cloudant package using your credentials. Binding a package allows you to set the default parameters that are inherited by every action and feed in the package. 

```bash
wsk package bind /whisk.system/cloudant "openwhisk-cloudant" \
  --param username "$CLOUDANT_USERNAME" \
  --param password "$CLOUDANT_PASSWORD" \
  --param host "$CLOUDANT_USERNAME.cloudant.com"
```

Triggers are a named channel for a class of event and can be explicitly fired by a user or fired on behalf of a user by an external event source, such as a feed. Use the code below to create a trigger to fire events when data is inserted into the "cats" database using the "changes" feed provided in the Cloudant package we just bound.
```bash
wsk trigger create data-inserted-trigger \
  --feed "/_/openwhisk-cloudant/changes" \
  --param dbname "cats"
```

# 2. Create OpenWhisk actions
Create a file named `process-change.js`. This file will define an OpenWhisk action written as a JavaScript function. This function will print out data that is written to Cloudant. For this example, we are expecting a cat with fields `name` and `color`.

```javascript
function main(params) {

  return new Promise(function(resolve, reject) {
    console.log(params.name);
    console.log(params.color);

    if (!params.name) {
      console.error('name parameter not set.');
      reject({
        'error': 'name parameter not set.'
      });
      return;
    } else {
      var message = 'A ' + params.color + ' cat named ' + params.name + ' was added.';
      console.log(message);
      resolve({
        change: message
      });
      return;
    }

  });

}
```

## Create action sequence and map to trigger
Create an OpenWhisk action from the JavaScript function that we just created.
```bash
wsk action create process-change process-change.js
```
OpenWhisk actions are stateless code snippets that can be invoked explicitly or in response to an event. To verify the creation of our action, invoke the action explicitly using the code below and pass the parameters using the --param command line argument.
```bash
wsk action invoke \
  --blocking \
  --param name Tahoma \
  --param color Tabby \
  process-change
```
Chain together multiple actions using a sequence. Here we will connect the cloudant "read" action with the "process-change" action we just created. The parameters (`name` and `color`) outputed from the cloudant "read" action will be passed automatically into our "process-change" action. 
``` bash
wsk action create process-change-cloudant-sequence \
  --sequence /_/openwhisk-cloudant/read,process-change
```

Rules map triggers with actions. Create a rule that maps the database change trigger to the sequence we just created. Once this rule is created, the actions (or sequence of actions) will be executed whenever the trigger is fired in response to new data inserted into the cloudant database. 
```bash
wsk rule create log-change-rule data-inserted-trigger process-change-cloudant-sequence
```

## Enter data to fire a change
Begin streaming the OpenWhisk activation log.
```bash
wsk activation poll
```

In the Cloudant dashboard, create a new document in the "cats" database. The `_id` will be created by Cloudant. Add the "name" and "color" attributes.
```json
{
  "_id": "xxxxxxxxxxxxxxxxxxxxxxxx",
  "name": "Tahoma",
  "color": "Tabby"
}
```

View the OpenWhisk log to look for the change notification.

# 3. Clean up
## Remove the API mappings and delete the actions

```bash
# Remove rule
wsk rule disable log-change-rule
wsk rule delete log-change-rule

# Remove trigger
wsk trigger delete data-inserted-trigger

# Remove actions
wsk action delete process-change-cloudant-sequence
wsk action delete process-change

# Remove package
wsk package delete "openwhisk-cloudant"
```

# Troubleshooting
Check for errors first in the OpenWhisk activation log. Tail the log on the command line with `wsk activation poll` or drill into details visually with the [monitoring console on Bluemix](https://console.ng.bluemix.net/openwhisk/dashboard).

If the error is not immediately obvious, make sure you have the [latest version of the `wsk` CLI installed](https://console.ng.bluemix.net/openwhisk/learn/cli). If it's older than a few weeks, download an update.
```bash
wsk property get --cliversion
```

# License
[Apache 2.0](LICENSE.txt)
