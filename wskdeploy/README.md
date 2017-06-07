# Whisk Deploy - Cloudant Trigger 

Download latest `wskdeploy` from the [release page] (https://github.com/apache/incubator-openwhisk-wskdeploy/releases) of [openwhisk-wskdeploy](https://github.com/apache/incubator-openwhisk-wskdeploy) project.

## Pre-requisites:

* Cloudant database instance named `openwhisk-cloudant` in Bluemix.
* Database named `cats` in the cloudant instance.
* Set these environment variables:

```
export CLOUDANT_INSTANCE="openwhisk-cloudant"
export CLOUDANT_USERNAME=""
export CLOUDANT_PASSWORD=""
export CLOUDANT_HOSTNAME="$CLOUDANT_USERNAME.cloudant.com"
export CLOUDANT_DATABASE="cats"
```

## Deployment:

Clone Cloudant Trigger repo and run wskdeploy:

```
git clone https://github.com/IBM/openwhisk-cloudant-trigger.git
wskdeploy -p openwhisk-cloudant-trigger/wskdeploy
```

## Undeploy:

Run `wskdeploy` to undeploy cloudant trigger:

```
wskdeploy undeploy -p openwhisk-cloudant-trigger/wskdeploy
```
