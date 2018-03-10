# Deploy step-by-step with the `bx` command line tool

## Prerequisites

You should have a basic understanding of the Cloud Functions/OpenWhisk programming model. If not, [try the action, trigger, and rule demo first](https://github.com/IBM/openwhisk-action-trigger-rule).

Also, you'll need an IBM Cloud account and the latest [`bx` command line tool with the Cloud Functions plugin installed and on your PATH](https://console.bluemix.net/openwhisk/learn/cli).

As an alternative to this end-to-end example, you might also consider the more [basic "building block" version](https://github.com/IBM/ibm-cloud-functions-message-hub-trigger) of this sample.

## Steps

1. [Provision Cloudant](#1-provision-cloudant)
2. [Create OpenWhisk actions, triggers, and rules](#2-create-openwhisk-actions-triggers-and-rules)
3. [Test database change events](#3-test-database-change-events)
4. [Delete actions, triggers, and rules](#4-delete-actions-triggers-and-rules)
5. [Recreate deployment manually](#5-recreate-deployment-manually)

## 1. Provision Cloudant

Log into the IBM Cloud, provision a [Cloudant database instance](https://console.ng.bluemix.net/catalog/services/cloudant-nosql-db/), and name it `openwhisk-cloudant`. Log into the Cloudant web console and create a database named `cats`.

Copy `template.local.env` to a new file named `local.env` and update the `CLOUDANT_INSTANCE`, `CLOUDANT_DATABASE`, `CLOUDANT_USERNAME`, and `CLOUDANT_PASSWORD` for your instance.

> **Note**: The Cloudant service credentials and connectivity details can be automatically bound to the OpenWhisk context with `wsk package refresh`, as they are in the [simpler version of this app](https://github.com/IBM/openwhisk-cloudant-trigger). However, since we are writing more than a simple JSON object back to Cloudant, we will use the Cloudant NPM client directly with the credentials, rather than through the Cloudant packaged write action.

## 2. Create OpenWhisk actions, triggers, and rules

`deploy.sh` is a convenience script reads the environment variables from `local.env` and creates the OpenWhisk actions, triggers, and rules on your behalf. Later you will run these commands yourself.

```bash
./deploy.sh --install
```

> **Note**: If you see any error messages, refer to the [Troubleshooting](#troubleshooting) section below.

## 3. Test database change events

To test, invoke the first action manually. Open one terminal window to poll the logs:

```bash
bx wsk activation poll
```

And in a second terminal, invoke the action:

```bash
bx wsk action invoke --blocking --result data-processing-cloudant/write-to-cloudant
```

## 4. Delete actions, triggers, and rules

Use `deploy.sh` again to tear down the OpenWhisk actions, triggers, and rules. You will recreate them step-by-step in the next section.

```bash
./deploy.sh --uninstall
```

## 5. Recreate deployment manually

This section provides a deeper look into what the `deploy.sh` script executes so that you understand how to work with OpenWhisk triggers, actions, rules, and packages in more detail.

### 5.1 Bind Cloudant package with credential parameters

Make the Cloudant instance on the IBM Cloud available as an event source.

```bash
bx wsk package refresh
```

### 5.2 Create trigger to fire events when data is inserted

Create a trigger named `image-uploaded` that subscribes to that Cloudant instance and specific database change events.

```bash
bx wsk trigger create image-uploaded \
  --feed "/_/Bluemix_${CLOUDANT_INSTANCE}_Credentials-1/changes" \
  --param dbname "$CLOUDANT_DATABASE"
```

### 5.3 Create action that is manually invoked to write to database

Upload the action code named `write-to-cloudant` that inserts text and an image to Cloudant which will initiate a database change event.

```bash
bx wsk package create data-processing-cloudant
bx wsk action create data-processing-cloudant/write-to-cloudant ../runtimes/nodejs/actions/write-to-cloudant.js \
  --param CLOUDANT_USERNAME "$CLOUDANT_USERNAME" \
  --param CLOUDANT_PASSWORD "$CLOUDANT_PASSWORD" \
  --param CLOUDANT_DATABASE "$CLOUDANT_DATABASE"
```

### 5.4 Create action to respond to database insertions

Upload the action code named `write-from-cloudant` that will receive database change information and log it to the console.

```bash
bx wsk action create data-processing-cloudant/write-from-cloudant ../runtimes/nodejs/actions/write-from-cloudant.js
```

### 5.5 Create sequence that ties database read to handling action

Specify a linkage between the built-in Cloudant `read` action and the custom `write-from-cloudant` above in a sequence named `write-from-cloudant-sequence`.

```bash
bx wsk action create data-processing-cloudant/write-from-cloudant-sequence \
  --sequence /_/Bluemix_${CLOUDANT_INSTANCE}_Credentials-1/read,data-processing-cloudant/write-from-cloudant
```

### 5.6 Create rule that maps database change trigger to sequence

Declare a rule named `echo-images` that maps the trigger `image-uploaded` to the action sequence `write-from-cloudant-sequence`. Without this mapping, the trigger will fire but no logic will be executed in response.

```bash
bx wsk rule create echo-images image-uploaded data-processing-cloudant/write-from-cloudant-sequence
```

### 5.7 Test it again

```bash
bx wsk action invoke --blocking --result data-processing-cloudant/write-to-cloudant
```

## Troubleshooting

Check for errors first in the OpenWhisk activation log. Tail the log on the command line with `wsk activation poll` or drill into details visually with the [monitoring console on the IBM Cloud](https://console.ng.bluemix.net/openwhisk/dashboard).

If the error is not immediately obvious, make sure you have the [latest version of the `wsk` CLI installed](https://console.ng.bluemix.net/openwhisk/learn/cli). If it's older than a few weeks, download an update.

```bash
bx wsk property get --cliversion
```

## License

[Apache 2.0](LICENSE.txt)