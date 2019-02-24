# serverless-import-apigateway

[![Build Status](https://travis-ci.org/MikeSouza/serverless-import-apigateway.svg?branch=master)](https://travis-ci.org/MikeSouza/serverless-import-apigateway)
[![Coverage Status](https://coveralls.io/repos/github/MikeSouza/serverless-import-apigateway/badge.svg?branch=master)](https://coveralls.io/github/MikeSouza/serverless-import-apigateway?branch=master)

Dynamically import an existing AWS API Gateway into your Serverless stack.

## Purpose

This plugin allows you to specify the name and paths of your existing API Gateway's REST API and it will lookup and configure the provider's API Gateway with the necessary IDs.

By default, [Serverless](https://https://serverless.com) creates automatically an API Gateway for each Serverless stack or service (i.e. `serverless.yml`) you deploy. This is sufficient if you only have a single service and monolithic `serverless.yml` or can deploy your entire Serverless app at once, but if you wish to break up your monolithic Serverless app into multiple `serverless.yml` services and deploy each stack independently but share the same API Gateway stage/ REST API, then you want each service to use the same API Gateway.

Suppose you have the following Serverless service with an API Gateway created for a root endpoint `/`:

```yaml
service: first-service

provider:
  name: aws
  runtime: nodejs8.10

functions:
  submit:
    handler: handler.status
    events:
      - http:
          path: /
          method: GET

resources:
  Outputs:
    ExportedApiGatewayRestApi:
      Description: First service's API Gateway REST API resource ID
      Value:
        Ref: ApiGatewayRestApi # Logical ID
      Export:
        Name: ExportedApiGatewayRestApi
```

Now if you want to create another service and add additional endpoints to this existing API Gateway, Serverless supports [configuring](https://serverless.com/framework/docs/providers/aws/guide/serverless.yml) the AWS provider to have use an existing API Gateway with something like the following:

```yaml
service: second-service

provider:
  name: aws
  runtime: nodejs8.10
  apiGateway: # Optional API Gateway global config
    restApiId: 2kd8204f8d # REST API resource ID. Default is generated by the framework
    restApiRootResourceId: 9df5ik7fyy # Root resource ID, represent as / path
```

However, it's not ideal to hardcode the `restApiId` and `restApiRootResourceId` into your `serverless.yml`, which can be obtained via the AWS CLI with the [get-rest-apis](https://docs.aws.amazon.com/cli/latest/reference/apigateway/get-rest-apis.html) and [get-resources](https://docs.aws.amazon.com/cli/latest/reference/apigateway/get-resources.html) commands, respectively.

Unfortunately you cannot import the first service's exported CloudFormation stack output variable `ExportedApiGatewayRestApi` from within the provider's configuration using something like `restApiId: { 'Fn::ImportValue': ExportedApiGatewayRestApi }`. This will produce an error.

The next best thing is to write a shell script that queries these IDs with the AWS CLI by filtering on the REST API's name and paths, and populating environment variables or passing in parameters to the Serverless CLI, so that you can reference them from the provider's configuration with `${opt:restApiId}` or `${env:restApiId}`. The script might look something like this:

```bash
#!/usr/bin/env bash

STAGE=${1:-dev}
REGION=${2:-us-east-1}

RESTAPI_ID=$(aws apigateway get-rest-apis --region ${REGION} \
  --query "items[?name=='${STAGE}-first-service'].id" --output text)
ROOT_RESOURCE_ID=$(aws apigateway get-resources --region ${REGION} --rest-api-id ${RESTAPI_ID} \
  --query "items[?path=='/'].id" --output text)

sls deploy --restApiId ${RESTAPI_ID} --restApiRootResourceId ${ROOT_RESOURCE_ID}
```

This is still a hacky and less than ideal solution.

Enter this plugin- you can accomplish all of this and more *without* a shell script gluing things together!

## Install

`npm install serverless-import-apigateway --save-dev`

## Configuration

Add the plugin to your `serverless.yml`:

```yaml
plugins:
  - serverless-import-apigateway
```

Add the custom configuration:

```yaml
custom:
  importApiGateway:
    name: ${self:provider.stage}-existing-service  # Required
    path: /                                        # Optional
    resources:                                     # Optional
      - /existing
      - /existing/resource
```

| Property    | Required | Type     | Default | Description                                                |
|-------------|----------|----------|---------|------------------------------------------------------------|
| `name`      |  `true`  | `string` |         | The name of the REST API for the AWS API Gateway to import |
| `path`      |  `false` | `string` |   `/`   | The root resource path to import from the REST API         |
| `resources` |  `false` |  `array` |   `[]`  | The existing resource paths to import from the REST API    |

## Usage

Configuration of your `serverless.yml` is all you need.

There are no custom commands, just run: `sls deploy`
