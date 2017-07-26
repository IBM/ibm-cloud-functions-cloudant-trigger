# OpenWhisk building block - Cloudant Trigger
Create Cloudant data processing apps with Apache OpenWhisk on IBM Bluemix. This tutorial takes less than 5 minutes to complete. After this, move on to more complex serverless applications such as those tagged [_openwhisk-hands-on-demo_](https://github.com/search?q=topic%3Aopenwhisk-hands-on-demo+org%3AIBM&type=Repositories).

![Sample Architecture](https://openwhisk-ui-prod.cdn.us-south.s-bluemix.net/openwhisk/ngow-public/img/getting-started-database-changes.svg)

If you're not familiar with the OpenWhisk programming model [try the action, trigger, and rule sample first](https://github.com/IBM/openwhisk-action-trigger-rule). [You'll need a Bluemix account and the latest OpenWhisk command line tool](https://github.com/IBM/openwhisk-action-trigger-rule/blob/master/docs/OPENWHISK.md).

This example shows how to create an action that can be integrated with the built in Cloudant changes trigger and read action to execute logic when new data is added.

1. [Configure Cloudant](#1-configure-cloudant)
2. [Create OpenWhisk actions](#2-create-openwhisk-actions)
3. [Clean up](#3-clean-up)

# 1. Configure Cloudant
## Provision a Cloudant service instance
Log into Bluemix, create a [Cloudant database instance](https://console.ng.bluemix.net/catalog/services/cloudant-nosql-db/), and name it `openwhisk-cloudant`. Launch the Cloudant web console and create a database named `cats`. Set the corresponding names as environment variables in a terminal window:

```bash
export CLOUDANT_INSTANCE="openwhisk-cloudant"
export CLOUDANT_DATABASE="cats"
```

## Import the service credentials into the OpenWhisk environment
We will make use of the built-in [OpenWhisk Cloudant package](https://github.com/apache/incubator-openwhisk-package-cloudant), which contains a set of actions and feeds that integrate with both Apache Kafka and IBM Message Hub (based on Kafka).

On Bluemix, this package can be automatically configured with the credentials and connection information from the Message Hub instance we provisioned above. We make it available by refreshing our list of packages.

```bash
# Ensures the Cloudant credentials are available to OpenWhisk.
wsk package refresh
```

# 2. Create OpenWhisk actions, triggers, and rules
## Attach a trigger to the Cloudant database
Triggers can be explicitly fired by a user or fired on behalf of a user by an external event source, such as a feed. Use the code below to create a trigger to fire events when data is inserted into the "cats" database using the "changes" feed provided in the Cloudant package.
```bash
wsk trigger create data-inserted-trigger \
  --feed Bluemix_${CLOUDANT_INSTANCE}_Credentials-1/changes \
  --param dbname "$CLOUDANT_DATABASE"
```

## Create an action to process changes to the database
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

Create an OpenWhisk action from the JavaScript function.
```bash
wsk action create process-change process-change.js
```

To verify the creation of our action and unit test its logic, invoke the action explicitly using the code below and pass the parameters using the `--param` command line arguments.
```bash
wsk action invoke \
  --blocking \
  --param name Tahoma \
  --param color Tabby \
  process-change
```

## Create an action sequence and map to the trigger with a rule
We now chain together multiple actions using a sequence. Here we will connect the packaged Cloudant "read" action with the "process-change" action we just created. The parameters (`name` and `color`) output from the cloudant "read" action will be passed automatically to our "process-change" action.
``` bash
wsk action create process-change-cloudant-sequence \
  --sequence Bluemix_${CLOUDANT_INSTANCE}_Credentials-1/read,process-change
```

Rules map triggers to actions. Create a rule that maps the database change trigger to the sequence we just created. Once this rule is created, the actions (or sequence of actions) will be executed whenever the trigger is fired in response to new data inserted into the Cloudant database.
```bash
wsk rule create log-change-rule data-inserted-trigger process-change-cloudant-sequence
```

## Enter data to fire a change
Begin streaming the OpenWhisk activation log in a second terminal window.
```bash
wsk activation poll
```

In the Cloudant dashboard linked from the Bluemix console, create a new document in the "cats" database.
```json
{
  "name": "Tarball",
  "color": "Black"
}
```

View the OpenWhisk log to look for the change notification. You should see activation records for the rule, the trigger, the sequence, and the actions.

# 3. Clean up
## Remove the actions, triggers, rules, and package

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
wsk package delete Bluemix_${CLOUDANT_INSTANCE}_Credentials-1
```
# Automation

You can use an OpenWhisk deployment automation tool to automate deployment of Cloudant Trigger. Please refer to [README](wskdeploy/README.md) file.

# Troubleshooting
Check for errors first in the OpenWhisk activation log. Tail the log on the command line with `wsk activation poll` or drill into details visually with the [monitoring console on Bluemix](https://console.ng.bluemix.net/openwhisk/dashboard).

If the error is not immediately obvious, make sure you have the [latest version of the `wsk` CLI installed](https://console.ng.bluemix.net/openwhisk/learn/cli). If it's older than a few weeks, download an update.
```bash
wsk property get --cliversion
```

# License
[Apache 2.0](LICENSE.txt)
