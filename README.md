[![serverless](http://public.serverless.com/badges/v3.svg)](http://www.serverless.com) [![Build Status](https://travis-ci.org/horike37/serverless-step-functions.svg?branch=master)](https://travis-ci.org/horike37/serverless-step-functions) [![npm version](https://badge.fury.io/js/serverless-step-functions.svg)](https://badge.fury.io/js/serverless-step-functions) [![Coverage Status](https://coveralls.io/repos/github/horike37/serverless-step-functions/badge.svg?branch=master)](https://coveralls.io/github/horike37/serverless-step-functions?branch=master) [![MIT License](http://img.shields.io/badge/license-MIT-blue.svg?style=flat)](LICENSE) [![serverless-step-functions Dev Token](https://badge.devtoken.rocks/serverless-step-functions)](https://devtoken.rocks/package/serverless-step-functions)
# Serverless Step Functions
This is the Serverless Framework plugin for AWS Step Functions.

## Install
Run `npm install` in your Serverless project.
```
$ npm install --save-dev serverless-step-functions
```

Add the plugin to your serverless.yml file
```yml
plugins:
  - serverless-step-functions
```

## Setup
Specifies your statemachine definition using Amazon States Language in a `definition` statement in serverless.yml.
We recommend to use [serverless-pseudo-parameters](https://www.npmjs.com/package/serverless-pseudo-parameters) plugin together so that it makes it easy to set up `Resource` section under `definition`.

```yml
functions:
  hellofunc:
    handler: handler.hello

stepFunctions:
  stateMachines:
    hellostepfunc1:
      events:
        - http:
            path: gofunction
            method: GET
        - schedule:
            rate: rate(10 minutes)
            enabled: true
            input:
              key1: value1
              key2: value2
              stageParams:
                stage: dev
      name: myStateMachine
      definition:
        Comment: "A Hello World example of the Amazon States Language using an AWS Lambda Function"
        StartAt: HelloWorld1
        States:
          HelloWorld1:
            Type: Task
            Resource: arn:aws:lambda:#{AWS::Region}:#{AWS::AccountId}:function:${self:service}-${opt:stage}-hello
            End: true
      dependsOn: CustomIamRole
      alarms:
        topics:
          ok: arn:aws:sns:us-east-1:1234567890:NotifyMe
          alarm: arn:aws:sns:us-east-1:1234567890:NotifyMe
          insufficientData: arn:aws:sns:us-east-1:1234567890:NotifyMe
        metrics:
          - executionsTimeOut
          - executionsFailed
          - executionsAborted
          - metric: executionThrottled
            treatMissingData: breaching # overrides below default
        treatMissingData: ignore # optional
    hellostepfunc2:
      definition:
        StartAt: HelloWorld2
        States:
          HelloWorld2:
            Type: Task
            Resource: arn:aws:states:#{AWS::Region}:#{AWS::AccountId}:activity:myTask
            End: true
      dependsOn:
        - DynamoDBTable
        - KinesisStream
        - CUstomIamRole
  activities:
    - myTask
    - yourTask

plugins:
  - serverless-step-functions
  - serverless-pseudo-parameters
```

### Adding a custom name for a stateMachine
In case you need to interpolate a specific stage or service layer variable as the
stateMachines name you can add a `name` property to your yaml.

```yml
service: messager

functions:
  sendMessage:
    handler: handler.sendMessage

stepFunctions:
  stateMachines:
    sendMessageFunc:
      name: sendMessageFunc-${self:custom.service}-${opt:stage}
      definition:
        <your definition>

plugins:
  - serverless-step-functions
```

### Adding a custom logical id for a stateMachine
You can use a custom logical id that is only unique within the stack as opposed to the name that needs to be unique globally. This can make referencing the state machine easier/simpler because you don't have to duplicate the interpolation logic everywhere you reference the state machine.

```yml
service: messager

functions:
  sendMessage:
    handler: handler.sendMessage

stepFunctions:
  stateMachines:
    sendMessageFunc:
      id: SendMessageStateMachine
      name: sendMessageFunc-${self:custom.service}-${opt:stage}
      definition:
        <your definition>

plugins:
  - serverless-step-functions
```

You can then `Ref: SendMessageStateMachine` in various parts of CloudFormation or serverless.yml

#### Depending on another logical id
If your state machine depends on another resource defined in your `serverless.yml` then you can add a `dependsOn` field to the state machine `definition`. This would add the `DependsOn`clause to the generated CloudFormation template.

This `dependsOn` field can be either a string, or an array of strings.

```yaml
stepFunctions:
  stateMachines:
    myStateMachine:
      dependsOn: myDB

    myOtherStateMachine:
      dependsOn:
        - myOtherDB
        - myStream
```

#### CloudWatch Alarms
It's common practice to want to monitor the health of your state machines and be alerted when something goes wrong. You can either:

* do this using the [serverless-plugin-aws-alerts](https://github.com/ACloudGuru/serverless-plugin-aws-alerts), which lets you configure custom CloudWatch Alarms against the various metrics that Step Functions publishes.
* or, you can use the built-in `alarms` configuration from this plugin, which gives you an opinionated set of default alarms (see below)

```yaml
stepFunctions:
  stateMachines:
    myStateMachine:
      alarms:
        topics:
          ok: arn:aws:sns:us-east-1:1234567890:NotifyMe
          alarm: arn:aws:sns:us-east-1:1234567890:NotifyMe
          insufficientData: arn:aws:sns:us-east-1:1234567890:NotifyMe
        metrics:
          - executionsTimeOut
          - executionsFailed
          - executionsAborted
          - executionThrottled
        treatMissingData: missing
```

Both `topics` and `metrics` are required properties. There are 4 supported metrics, each map to the CloudWatch Metrics that Step Functions publishes for your executions.

You can configure how the CloudWatch Alarms should treat missing data:

* `missing` (AWS default): The alarm does not consider missing data points when evaluating whether to change state.
* `ignore`: The current alarm state is maintained.
* `breaching`: Missing data points are treated as breaching the threshold.
* `notBreaching`: Missing data points are treated as being within the threshold.

For more information, please refer to the [official documentation](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/AlarmThatSendsEmail.html#alarms-and-missing-data).

The generated CloudWatch alarms would have the following configurations:
```yaml
namespace: 'AWS/States'
metric: <ExecutionsTimeOut | ExecutionsFailed | ExecutionsAborted | ExecutionThrottled>
threshold: 1
period: 60
evaluationPeriods: 1
ComparisonOperator: GreaterThanOrEqualToThreshold
Statistic: Sum
treatMissingData: <missing (default) | ignore | breaching | notBreaching>
Dimensions:
  - Name: StateMachineArn
    Value: <ArnOfTheStateMachine>
```

You can also override the default `treatMissingData` setting for a particular alarm by specifying an override:

```yml
alarms:
  topics:
    ok: arn:aws:sns:us-east-1:1234567890:NotifyMe
    alarm: arn:aws:sns:us-east-1:1234567890:NotifyMe
    insufficientData: arn:aws:sns:us-east-1:1234567890:NotifyMe
  metrics:
    - executionsTimeOut
    - executionsFailed
    - executionsAborted
    - metric: executionThrottled
      treatMissingData: breaching # override
  treatMissingData: ignore # default
```

#### Current Gotcha
Please keep this gotcha in mind if you want to reference the `name` from the `resources` section. To generate Logical ID for CloudFormation, the plugin transforms the specified name in serverless.yml based on the following scheme.

- Transform a leading character into uppercase
- Transform `-` into Dash
- Transform `_` into Underscore

If you want to use variables system in name statement, you can't put the variables as a prefix like this:`${self:service}-${opt:stage}-myStateMachine` since the variables are transformed within Output section, as a result, the reference will be broken.

The correct sample is here.

```yaml
stepFunctions:
  stateMachines:
    myStateMachine:
      name: myStateMachine-${self:service}-${opt:stage}
...

resources:
  Outputs:
    myStateMachine:
      Value:
        Ref: MyStateMachineDash${self:service}Dash${opt:stage}
```

## Events
### API Gateway
To create HTTP endpoints as Event sources for your StepFunctions statemachine

#### Simple HTTP Endpoint
This setup specifies that the hello statemachine should be run when someone accesses the API gateway at hello via a GET request.

Here's an example:

```yml
stepFunctions:
  stateMachines:
    hello:
      events:
        - http:
            path: hello
            method: GET
      definition:
```
#### HTTP Endpoint with Extended Options

Here You can define an POST endpoint for the path posts/create.

```yml
stepFunctions:
  stateMachines:
    hello:
      events:
        - http:
            path: posts/create
            method: POST
      definition:
```

#### Share API Gateway and API Resources

You can [share the same API Gateway](https://serverless.com/framework/docs/providers/aws/events/apigateway/#share-api-gateway-and-api-resources) between multiple projects by referencing its REST API ID and Root Resource ID in serverless.yml as follows:

```yml
service: service-name
provider:
  name: aws
  apiGateway:
    # REST API resource ID. Default is generated by the framework
    restApiId: xxxxxxxxxx
    # Root resource, represent as / path
    restApiRootResourceId: xxxxxxxxxx

functions:
  ...
```
If your application has many nested paths, you might also want to break them out into smaller services.

However, Cloudformation will throw an error if we try to generate an existing path resource. To avoid that, we reference the resource ID:

```yml
service: service-a
provider:
  apiGateway:
    restApiId: xxxxxxxxxx
    restApiRootResourceId: xxxxxxxxxx
    # List of existing resources that were created in the REST API. This is required or the stack will be conflicted
    restApiResources:
      /users: xxxxxxxxxx

functions:
  ...
```

Now we can define endpoints using existing API Gateway ressources

```yml
stepFunctions:
  stateMachines:
    hello:
      events:
        - http:
            path: users/create
            method: POST
```

#### Enabling CORS

To set CORS configurations for your HTTP endpoints, simply modify your event configurations as follows:

```yml
stepFunctions:
  stateMachines:
    hello:
      events:
        - http:
            path: posts/create
            method: POST
            cors: true
      definition:
```

Setting cors to true assumes a default configuration which is equivalent to:

```yml
stepFunctions:
  stateMachines:
    hello:
      events:
        - http:
            path: posts/create
            method: POST
            cors:
              origin: '*'
              headers:
                - Content-Type
                - X-Amz-Date
                - Authorization
                - X-Api-Key
                - X-Amz-Security-Token
                - X-Amz-User-Agent
              allowCredentials: false
      definition:
```

Configuring the cors property sets Access-Control-Allow-Origin, Access-Control-Allow-Headers, Access-Control-Allow-Methods,Access-Control-Allow-Credentials headers in the CORS preflight response.
To enable the Access-Control-Max-Age preflight response header, set the maxAge property in the cors object:

```yml
stepFunctions:
  stateMachines:
    SfnApiGateway:
      events:
        - http:
            path: /playground/start
            method: post
            cors:
              origin: '*'
              maxAge: 86400
```

#### HTTP Endpoints with AWS_IAM Authorizers

If you want to require that the caller submit the IAM user's access keys in order to be authenticated to invoke your Lambda Function, set the authorizer to AWS_IAM as shown in the following example:

```yml
stepFunctions:
  stateMachines:
    hello:
      events:
        - http:
            path: posts/create
            method: POST
            authorizer: aws_iam
      definition:
```

#### HTTP Endpoints with Custom Authorizers

[Custom Authorizers](https://serverless.com/framework/docs/providers/aws/events/apigateway/#http-endpoints-with-custom-authorizers) allow you to run an AWS Lambda Function before your targeted AWS Lambda Function. This is useful for Microservice Architectures or when you simply want to do some Authorization before running your business logic.

You can enable Custom Authorizers for your HTTP endpoint by setting the Authorizer in your http event to another function in the same service, as shown in the following example:

```yml
stepFunctions:
  stateMachines:
    hello:
      - http:
          path: posts/create
          method: post
          authorizer: authorizerFunc
      definition:
```

If the Authorizer function does not exist in your service but exists in AWS, you can provide the ARN of the Lambda function instead of the function name, as shown in the following example:

```yml
stepFunctions:
  stateMachines:
    hello:
      - http:
          path: posts/create
          method: post
          authorizer: xxx:xxx:Lambda-Name
      definition:
```

### Share Authorizer

Auto-created Authorizer is convenient for conventional setup. However, when you need to define your custom Authorizer, or use COGNITO_USER_POOLS authorizer with shared API Gateway, it is painful because of AWS limitation. Sharing Authorizer is a better way to do.

```yml
stepFunctions:
  stateMachines:
    createUser:
      ...
      events:
        - http:
            path: /users
            ...
            authorizer:
              # Provide both type and authorizerId
              type: COGNITO_USER_POOLS # TOKEN, CUSTOM or COGNITO_USER_POOLS, same as AWS Cloudformation documentation
              authorizerId:
                Ref: ApiGatewayAuthorizer  # or hard-code Authorizer ID
```


#### Customizing request body mapping templates

The plugin generates default body mapping templates for `application/json` and `application/x-www-form-urlencoded` content types. If you'd like to add more content types or customize the default ones, you can do so by including them in `serverless.yml`:

```yml
stepFunctions:
  stateMachines:
    hello:
      events:
        - http:
            path: posts/create
            method: POST
            request:
              template:
                application/json: |
                  #set( $body = $util.escapeJavaScript($input.json('$')) )
                  #set( $name = $util.escapeJavaScript($input.json('$.data.attributes.order_id')) )
                  {
                    "input": "$body",
                    "name": "$name",
                    "stateMachineArn":"arn:aws:states:#{AWS::Region}:#{AWS::AccountId}:stateMachine:processOrderFlow-${opt:stage}"
                  }
      name: processOrderFlow-${opt:stage}
      definition:
```

#### Send request to an API
You can input an value as json in request body, the value is passed as the input value of your statemachine

```
$ curl -XPOST https://xxxxxxxxx.execute-api.us-east-1.amazonaws.com/dev/posts/create -d '{"foo":"bar"}'
```

#### Setting API keys for your Rest API
You can specify a list of API keys to be used by your service Rest API by adding an apiKeys array property to the provider object in serverless.yml. You'll also need to explicitly specify which endpoints are private and require one of the api keys to be included in the request by adding a private boolean property to the http event object you want to set as private. API Keys are created globally, so if you want to deploy your service to different stages make sure your API key contains a stage variable as defined below. When using API keys, you can optionally define usage plan quota and throttle, using usagePlan object.

Here's an example configuration for setting API keys for your service Rest API:

```yml
service: my-service
provider:
  name: aws
  apiKeys:
    - myFirstKey
    - ${opt:stage}-myFirstKey
    - ${env:MY_API_KEY} # you can hide it in a serverless variable
  usagePlan:
    quota:
      limit: 5000
      offset: 2
      period: MONTH
    throttle:
      burstLimit: 200
      rateLimit: 100
functions:
  hello:
    handler: handler.hello

    stepFunctions:
      stateMachines:
        statemachine1:
          name: ${self:service}-${opt:stage}-statemachine1
          events:
            - http:
                path: /hello
                method: post
                private: true
          definition:
            Comment: "A Hello World example of the Amazon States Language using an AWS Lambda Function"
            StartAt: HelloWorld1
            States:
              HelloWorld1:
                Type: Task
                Resource: arn:aws:lambda:#{AWS::Region}:#{AWS::AccountId}:function:${self:service}-${opt:stage}-hello
                End: true


    plugins:
      - serverless-step-functions
      - serverless-pseudo-parameters
```

Please note that those are the API keys names, not the actual values. Once you deploy your service, the value of those API keys will be auto generated by AWS and printed on the screen for you to use. The values can be concealed from the output with the --conceal deploy option.

Clients connecting to this Rest API will then need to set any of these API keys values in the x-api-key header of their request. This is only necessary for functions where the private property is set to true.

### Schedule
The following config will attach a schedule event and causes the stateMachine `crawl` to be called every 2 hours. The configuration allows you to attach multiple schedules to the same stateMachine. You can either use the `rate` or `cron` syntax. Take a look at the [AWS schedule syntax documentation](http://docs.aws.amazon.com/AmazonCloudWatch/latest/events/ScheduledEvents.html) for more details.

```yaml
stepFunctions:
  stateMachines:
    crawl:
      events:
        - schedule: rate(2 hours)
        - schedule: cron(0 12 * * ? *)
      definition:
```

## Enabling / Disabling

**Note:** `schedule` events are enabled by default.

This will create and attach a schedule event for the `aggregate` stateMachine which is disabled. If enabled it will call
the `aggregate` stateMachine every 10 minutes.

```yaml
stepFunctions:
  stateMachines:
    aggregate:
      events:
        - schedule:
            rate: rate(10 minutes)
            enabled: false
            input:
              key1: value1
              key2: value2
              stageParams:
                stage: dev
        - schedule:
            rate: cron(0 12 * * ? *)
            enabled: false
            inputPath: '$.stageVariables'
```

## Specify Name and Description

Name and Description can be specified for a schedule event. These are not required properties.

```yaml
events:
  - schedule:
      name: your-scheduled-rate-event-name
      description: 'your scheduled rate event description'
      rate: rate(2 hours)
```

### CloudWatch Event
## Simple event definition

This will enable your Statemachine to be called by an EC2 event rule.
Please check the page of [Event Types for CloudWatch Events](http://docs.aws.amazon.com/AmazonCloudWatch/latest/events/EventTypes.html).

```yml
stepFunctions:
  stateMachines:
    first:
      events:
        - cloudwatchEvent:
            event:
              source:
                - "aws.ec2"
              detail-type:
                - "EC2 Instance State-change Notification"
              detail:
                state:
                  - pending
      definition:
        ...
```

## Enabling / Disabling

**Note:** `cloudwatchEvent` events are enabled by default.

This will create and attach a disabled `cloudwatchEvent` event for the `myCloudWatch` statemachine.

```yml
stepFunctions:
  stateMachines:
    cloudwatchEvent:
      events:
        - cloudwatchEvent:
            event:
              source:
                - "aws.ec2"
              detail-type:
                - "EC2 Instance State-change Notification"
              detail:
                state:
                  - pending
            enabled: false
      definition:
        ...
```

## Specify Input or Inputpath

You can specify input values ​​to the Lambda function.

```yml
stepFunctions:
  stateMachines:
    cloudwatchEvent:
      events:
        - cloudwatchEvent:
            event:
              source:
                - "aws.ec2"
              detail-type:
                - "EC2 Instance State-change Notification"
              detail:
                state:
                  - pending
            input:
              key1: value1
              key2: value2
              stageParams:
                stage: dev
        - cloudwatchEvent:
            event:
              source:
                - "aws.ec2"
              detail-type:
                - "EC2 Instance State-change Notification"
              detail:
                state:
                  - pending
            inputPath: '$.stageVariables'
      definition:
        ...
```

## Specifying a Description

You can also specify a CloudWatch Event description.

```yml
stepFunctions:
  stateMachines:
    cloudwatchEvent:
      events:
        - cloudwatchEvent:
            description: 'CloudWatch Event triggered on EC2 Instance pending state'
            event:
              source:
                - "aws.ec2"
              detail-type:
                - "EC2 Instance State-change Notification"
              detail:
                state:
                  - pending
      definition:
        ...
```

## Specifying a Name

You can also specify a CloudWatch Event name. Keep in mind that the name must begin with a letter; contain only ASCII letters, digits, and hyphens; and not end with a hyphen or contain two consecutive hyphens. More infomation [here](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-properties-name.html).

```yml
stepFunctions:
  stateMachines:
    cloudwatchEvent:
      events:
        - cloudwatchEvent:
            name: 'my-cloudwatch-event-name'
            event:
              source:
                - "aws.ec2"
              detail-type:
                - "EC2 Instance State-change Notification"
              detail:
                state:
                  - pending
      definition:
        ...
```


## Command
### deploy
Run `sls deploy`, the defined Stepfunctions are deployed.

### invoke
```
$ sls invoke stepf --name <stepfunctionname> --data '{"foo":"bar"}'
```

#### options

- --name or -n The name of the step function in your service that you want to invoke. Required.
- --stage or -s The stage in your service you want to invoke your step function.
- --region or -r The region in your stage that you want to invoke your step function.
- --data or -d String data to be passed as an event to your step function.
- --path or -p The path to a json file with input data to be passed to the invoked step function.

## IAM Role

The IAM roles required to run Statemachine are automatically generated. It is also possible to specify ARN directly.

Here's an example:

```yml
stepFunctions:
  stateMachines:
    hello:
      role: arn:aws:iam::xxxxxxxx:role/yourRole
      definition:
```

It is also possible to use the [CloudFormation intrinsic functions](https://docs.aws.amazon.com/en_en/AWSCloudFormation/latest/UserGuide/intrinsic-function-reference.html) to reference resources from elsewhere:

```yml
stepFunctions:
  stateMachines:
    hello:
      role:
        Ref: StateMachineRole
      definition:
        ...

resources:
  Resources:
    StateMachineRole:
      Type: AWS::IAM::Role
      Properties:
        ...
```

The short form of the intrinsic functions (i.e. `!Sub`, `!Ref`) is not supported at the moment.

## Tips
### How to specify the stateMachine ARN to environment variables
Here is serverless.yml sample to specify the stateMachine ARN to environment variables.
This makes it possible to trigger your statemachine through Lambda events

```yml
functions:
  hello:
    handler: handler.hello
    environment:
      statemachine_arn: ${self:resources.Outputs.MyStateMachine.Value}

stepFunctions:
  stateMachines:
    hellostepfunc:
      name: myStateMachine
      definition:
        <your definition>

resources:
  Outputs:
    MyStateMachine:
      Description: The ARN of the example state machine
      Value:
        Ref: MyStateMachine

plugins:
  - serverless-step-functions
```
## Sample statemachines setting in serverless.yml
### Wait State
``` yaml
functions:
  hello:
    handler: handler.hello

stepFunctions:
  stateMachines:
    yourWateMachine:
      definition:
        Comment: "An example of the Amazon States Language using wait states"
        StartAt: FirstState
        States:
          FirstState:
            Type: Task
            Resource: arn:aws:lambda:#{AWS::Region}:#{AWS::AccountId}:function:${self:service}-${opt:stage}-hello
            Next: wait_using_seconds
          wait_using_seconds:
            Type: Wait
            Seconds: 10
            Next: wait_using_timestamp
          wait_using_timestamp:
            Type: Wait
            Timestamp: '2015-09-04T01:59:00Z'
            Next: wait_using_timestamp_path
          wait_using_timestamp_path:
            Type: Wait
            TimestampPath: "$.expirydate"
            Next: wait_using_seconds_path
          wait_using_seconds_path:
            Type: Wait
            SecondsPath: "$.expiryseconds"
            Next: FinalState
          FinalState:
            Type: Task
            Resource: arn:aws:lambda:#{AWS::Region}:#{AWS::AccountId}:function:${self:service}-${opt:stage}-hello
            End: true
plugins:
  - serverless-step-functions
  - serverless-pseudo-parameters
```

### Retry Failture
``` yaml
functions:
  hello:
    handler: handler.hello

stepFunctions:
  stateMachines:
    yourRetryMachine:
      definition:
        Comment: "A Retry example of the Amazon States Language using an AWS Lambda Function"
        StartAt: HelloWorld
        States:
          HelloWorld:
            Type: Task
            Resource: arn:aws:lambda:#{AWS::Region}:#{AWS::AccountId}:function:${self:service}-${opt:stage}-hello
            Retry:
            - ErrorEquals:
              - HandledError
              IntervalSeconds: 1
              MaxAttempts: 2
              BackoffRate: 2
            - ErrorEquals:
              - States.TaskFailed
              IntervalSeconds: 30
              MaxAttempts: 2
              BackoffRate: 2
            - ErrorEquals:
              - States.ALL
              IntervalSeconds: 5
              MaxAttempts: 5
              BackoffRate: 2
            End: true
plugins:
  - serverless-step-functions
  - serverless-pseudo-parameters
```

### Parallel

```yaml
functions:
  hello:
    handler: handler.hello

stepFunctions:
  stateMachines:
    yourParallelMachine:
      definition:
        Comment: "An example of the Amazon States Language using a parallel state to execute two branches at the same time."
        StartAt: Parallel
        States:
          Parallel:
            Type: Parallel
            Next: Final State
            Branches:
            - StartAt: Wait 20s
              States:
                Wait 20s:
                  Type: Wait
                  Seconds: 20
                  End: true
            - StartAt: Pass
              States:
                Pass:
                  Type: Pass
                  Next: Wait 10s
                Wait 10s:
                  Type: Wait
                  Seconds: 10
                  End: true
          Final State:
            Type: Pass
            End: true
plugins:
  - serverless-step-functions
  - serverless-pseudo-parameters
```

### Catch Failure

```yaml
functions:
  hello:
    handler: handler.hello

stepFunctions:
  stateMachines:
    yourCatchMachine:
      definition:
        Comment: "A Catch example of the Amazon States Language using an AWS Lambda Function"
        StartAt: HelloWorld
        States:
          HelloWorld:
            Type: Task
            Resource: arn:aws:lambda:#{AWS::Region}:#{AWS::AccountId}:function:${self:service}-${opt:stage}-hello
            Catch:
            - ErrorEquals: ["HandledError"]
              Next: CustomErrorFallback
            - ErrorEquals: ["States.TaskFailed"]
              Next: ReservedTypeFallback
            - ErrorEquals: ["States.ALL"]
              Next: CatchAllFallback
            End: true
          CustomErrorFallback:
            Type: Pass
            Result: "This is a fallback from a custom lambda function exception"
            End: true
          ReservedTypeFallback:
            Type: Pass
            Result: "This is a fallback from a reserved error code"
            End: true
          CatchAllFallback:
            Type: Pass
            Result: "This is a fallback from a reserved error code"
            End: true
plugins:
  - serverless-step-functions
  - serverless-pseudo-parameters
```

### Choice

```yaml
functions:
  hello1:
    handler: handler.hello1
  hello2:
    handler: handler.hello2
  hello3:
    handler: handler.hello3
  hello4:
    handler: handler.hello4

stepFunctions:
  stateMachines:
    yourChoiceMachine:
      definition:
        Comment: "An example of the Amazon States Language using a choice state."
        StartAt: FirstState
        States:
          FirstState:
            Type: Task
            Resource: arn:aws:lambda:${opt:region}:${self:custom.accountId}:function:${self:service}-${opt:stage}-hello1
            Next: ChoiceState
          ChoiceState:
            Type: Choice
            Choices:
            - Variable: "$.foo"
              NumericEquals: 1
              Next: FirstMatchState
            - Variable: "$.foo"
              NumericEquals: 2
              Next: SecondMatchState
            Default: DefaultState
          FirstMatchState:
            Type: Task
            Resource: arn:aws:lambda:#{AWS::Region}:#{AWS::AccountId}:function:${self:service}-${opt:stage}-hello2
            Next: NextState
          SecondMatchState:
            Type: Task
            Resource: arn:aws:lambda:#{AWS::Region}:#{AWS::AccountId}:function:${self:service}-${opt:stage}-hello3
            Next: NextState
          DefaultState:
            Type: Fail
            Cause: "No Matches!"
          NextState:
            Type: Task
            Resource: arn:aws:lambda:#{AWS::Region}:#{AWS::AccountId}:function:${self:service}-${opt:stage}-hello4
            End: true
plugins:
  - serverless-step-functions
  - serverless-pseudo-parameters
```
