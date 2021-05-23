# Project 3 - A Ridiculous Stretch Goal

Okay, this project uses my favorite feature of API Gateway and it's one that not too many people know how to use.
I'm giving you access to a super secret API that nobody is supposed to know about.

Previously, we used the SAM CLI to scaffold our application and then used the `Events:` property on the `AWS::Serverless::Function` resource to avoid having to write the whole API definition manually.
We're going to stop doing that so we get a better appreciation for what the SAM transform is doing on our behalf.
Let's start with a very simple template.

```yaml
AWSTemplateFormatVersion: '2010-09-09' # Supposedly, this is option if you use the YAML syntax for CFN
Resources:
    ExampleRestApi:
        Type: AWS::ApiGateway::RestApi
        Properties:
            Name: MyRESTAPI
```

But, an API by itself isn't that useful, so let's add a couple paths.

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Resources:
    ExampleRestApi:
        Type: AWS::ApiGateway::RestApi
        Properties:
            Name: MyRESTAPI

    FunctionsPath:
          Type: 'AWS::ApiGateway::Resource'
          Properties:
            RestApiId: !Ref ExampleRestApi
            ParentId: !GetAtt ExampleRestApi.RootResourceId
            PathPart: functions

    TablesPath:
          Type: 'AWS::ApiGateway::Resource'
          Properties:
            RestApiId: !Ref ExampleRestApi
            ParentId: !GetAtt ExampleRestApi.RootResourceId
            PathPart: tables
```

Even an API and a few paths aren't helpful if it can't process some HTTP requests, let's add support for `GET` requests.

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Resources:
    ExampleRestApi:
        Type: AWS::ApiGateway::RestApi
        Properties:
            Name: MyRESTAPI

    FunctionsPath:
          Type: 'AWS::ApiGateway::Resource'
          Properties:
            RestApiId: !Ref ExampleRestApi
            ParentId: !GetAtt ExampleRestApi.RootResourceId
            PathPart: functions

    ListFunctionsGetMethod:
        Type: AWS::ApiGateway::Method
        Properties:
            OperationName: "ListFunctions"
            HttpMethod: GET
            ResourceId: !Ref FunctionsPath
            RestApiId: !Ref ExampleRestApi
            AuthorizationType: NONE
            Integration:
                Type: MOCK
                IntegrationResponses:
                  - ResponseTemplates:
                        application/json: "{\"hello\": \"Ferris\"}"
                    SelectionPattern: '2\d{2}'
                    StatusCode: 200
                RequestTemplates:
                    application/json: "{\n \"statusCode\": 200\n}"
            MethodResponses:
              - StatusCode: 200
            

    TablesPath:
          Type: 'AWS::ApiGateway::Resource'
          Properties:
            RestApiId: !Ref ExampleRestApi
            ParentId: !GetAtt ExampleRestApi.RootResourceId
            PathPart: tables

    ListTablesGetMethod:
        Type: AWS::ApiGateway::Method
        Properties:
            OperationName: "ListTables"
            HttpMethod: GET
            ResourceId: !Ref TablesPath
            RestApiId: !Ref ExampleRestApi
            AuthorizationType: NONE
            Integration:
                Type: MOCK
                IntegrationResponses:
                  - ResponseTemplates:
                        application/json: "{\"hello\": \"Ferris\"}"
                    SelectionPattern: '2\d{2}'
                    StatusCode: 200
                RequestTemplates:
                    application/json: "{\n \"statusCode\": 200\n}"
            MethodResponses:
              - StatusCode: 200
```

Let's deploy this and make sure it works.

```text
aws cloudformation deploy --template-file ./template.yaml --stack-name my-artisinal-api
```

Now we've created an API with two paths that each support a GET operation.
You'll notice we didn't get a URL back that we could easily cURL to test that the API worked.
This is because there are two main steps in deploying an API Gateway REST API.
The first is creating the API definition and the second is creating a 'deployment stage'.
The AWS SAM transform handled this for us in the previous projects, but we'll have to do it ourselves now since we're not using a Lambda function anymore.

Let's add a 'deployment' of this API so that we can make requests to it.

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Resources:
    ExampleRestApi:
        Type: AWS::ApiGateway::RestApi
        Properties:
            Name: MyRESTAPI

    FunctionsPath:
          Type: 'AWS::ApiGateway::Resource'
          Properties:
            RestApiId: !Ref ExampleRestApi
            ParentId: !GetAtt ExampleRestApi.RootResourceId
            PathPart: functions

    ListFunctionsGetMethod:
        Type: AWS::ApiGateway::Method
        Properties:
            OperationName: "ListFunctions"
            HttpMethod: GET
            ResourceId: !Ref FunctionsPath
            RestApiId: !Ref ExampleRestApi
            AuthorizationType: NONE
            Integration:
                Type: MOCK
                IntegrationResponses:
                  - ResponseTemplates:
                        application/json: "{\"hello\": \"Ferris\"}"
                    SelectionPattern: '2\d{2}'
                    StatusCode: 200
                RequestTemplates:
                    application/json: "{\n \"statusCode\": 200\n}"
            MethodResponses:
              - StatusCode: 200
            

    TablesPath:
          Type: 'AWS::ApiGateway::Resource'
          Properties:
            RestApiId: !Ref ExampleRestApi
            ParentId: !GetAtt ExampleRestApi.RootResourceId
            PathPart: tables

    ListTablesGetMethod:
        Type: AWS::ApiGateway::Method
        Properties:
            OperationName: "ListTables"
            HttpMethod: GET
            ResourceId: !Ref TablesPath
            RestApiId: !Ref ExampleRestApi
            AuthorizationType: NONE
            Integration:
                Type: MOCK
                IntegrationResponses:
                  - ResponseTemplates:
                        application/json: "{\"hello\": \"Ferris\"}"
                    SelectionPattern: '2\d{2}'
                    StatusCode: 200
                RequestTemplates:
                    application/json: "{\n \"statusCode\": 200\n}"
            MethodResponses:
              - StatusCode: 200

    Deployment01:
      DependsOn:
        - ListTablesGetMethod
        - ListFunctionsGetMethod
      Type: 'AWS::ApiGateway::Deployment'
      Properties:
        RestApiId: !Ref ExampleRestApi
        Description: My deployment
        StageName: Prod

Outputs:
  HelloWorldApi:
    Description: "API Gateway endpoint URL for Prod stage"
    Value: !Sub "https://${ExampleRestApi}.execute-api.${AWS::Region}.amazonaws.com/Prod/"
```

We deploy the stack with:

```bash
aws cloudformation deploy --stack-name my-artisinal-api --template-file ./template.yaml
```

Once the deployment completes, we can get the url by querying CloudFormation for the outputs from the stack.

```bash
aws cloudformation describe-stacks --stack-name my-artisinal-api --query "Stacks[*].Outputs[?OutputKey=='HelloWorldApi'].OutputValue" --output text
```

Now we'll update the mocked DynamoDB ListTables tables command to use the actual API call.
In order for API Gateway to make the API call, we need to give it permissions via IAM.
You'll note the new `AWS::IAM::Role` resource at the end of the `Resources` section of the template.

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Resources:
    ExampleRestApi:
        Type: AWS::ApiGateway::RestApi
        Properties:
            Name: MyRESTAPI

    FunctionsPath:
          Type: 'AWS::ApiGateway::Resource'
          Properties:
            RestApiId: !Ref ExampleRestApi
            ParentId: !GetAtt ExampleRestApi.RootResourceId
            PathPart: functions

    ListFunctionsGetMethod:
        Type: AWS::ApiGateway::Method
        Properties:
            OperationName: "ListFunctions"
            HttpMethod: GET
            ResourceId: !Ref FunctionsPath
            RestApiId: !Ref ExampleRestApi
            AuthorizationType: NONE
            Integration:
                Type: MOCK
                IntegrationResponses:
                  - ResponseTemplates:
                        application/json: "{\"hello\": \"Ferris\"}"
                    SelectionPattern: '2\d{2}'
                    StatusCode: 200
                RequestTemplates:
                    application/json: "{\n \"statusCode\": 200\n}"
            MethodResponses:
              - StatusCode: 200
            

    TablesPath:
          Type: 'AWS::ApiGateway::Resource'
          Properties:
            RestApiId: !Ref ExampleRestApi
            ParentId: !GetAtt ExampleRestApi.RootResourceId
            PathPart: tables

    ListTablesGetMethod:
        Type: AWS::ApiGateway::Method
        Properties:
            OperationName: "ListTables"
            HttpMethod: GET
            ResourceId: !Ref TablesPath
            RestApiId: !Ref ExampleRestApi
            AuthorizationType: NONE
            # This property completely defines the integration with the AWS service that API Gateway will call when the API is invoked by clients.
            Integration:
              # Previously this was MOCK, now we're integrating with an AWS service so we change it to AWS.
              # This tells API Gateway that it needs to perform Sigv4 request signing on the API call it is about to make.
              Type: AWS
              # This is the HTTP method that API Gateway will use when calling the other AWS service. Almost all AWS API calls are POST requests.
              IntegrationHttpMethod: POST
              # This section defines how to transform the response we get back.
              # For example, we may get a bunch of metadata back but we just want to return the Table names.
              # This section is where we would define the transformation that strips out the metadata.
              IntegrationResponses:
              - StatusCode: 200
                ResponseTemplates:
                  application/json: ''
              # We'll cover this section more later, for now it's safe to treat it like the previous section, but for transforming the incoming request into a format that the back-end service understands.
              RequestTemplates:
                application/json: '{}'
              # This is used to define what API Gateway should do if it doesn't know how to perform one of the transforms defined in the previous section.
              PassthroughBehavior: WHEN_NO_TEMPLATES
              # These are dark magic and this blog post Iwrote a couple years ago does a decent job explaining what they do.
              # https://rboyd.dev/e41a775d-3dd6-4a9f-ab45-b01f3bddab83
              RequestParameters:
                integration.request.header.X-Amz-Target: "'DynamoDB_20120810.ListTables'"
                integration.request.header.Content-Type: "'application/x-amz-json-1.0'"
              # Again, dark magic.
              Uri: !Sub "arn:aws:apigateway:${AWS::Region}:dynamodb:path//"
              # This is the IAM Role to use for the request.
              # The Role needs to have apigateway.amazonaws.com as a principal that is trusted to assume the Role.
              Credentials: !GetAtt ListTablesRole.Arn
            MethodResponses:
              - StatusCode: 200

    Deployment02:
      DependsOn:
        - ListTablesGetMethod
        - ListFunctionsGetMethod
      Type: 'AWS::ApiGateway::Deployment'
      Properties:
        RestApiId: !Ref ExampleRestApi
        Description: My deployment
        StageName: Prod

    ListTablesRole:
      Type: 'AWS::IAM::Role'
      Description: |
        IAM Role needed by API Gateway in order to have the correct permissions needed to call other AWS APIs on your behalf.
        See more info at https://docs.aws.amazon.com/apigateway/latest/developerguide/permissions.html
      Properties:
        AssumeRolePolicyDocument:
          Version: "2012-10-17"
          Statement:
            - Effect: Allow
              Principal:
                Service:
                  - apigateway.amazonaws.com
              Action:
                - 'sts:AssumeRole'
        Path: /
        Policies:
          - PolicyName: root
            PolicyDocument:
              Version: "2012-10-17"
              Statement:
                - Effect: Allow
                  Action: 'dynamodb:ListTables'
                  Resource: '*'

Outputs:
  HelloWorldApi:
    Description: "API Gateway endpoint URL for Prod stage"
    Value: !Sub "https://${ExampleRestApi}.execute-api.${AWS::Region}.amazonaws.com/Prod/"
```

We'll deploy this template with the following command:

```bash
aws cloudformation deploy --stack-name my-artisinal-api --template-file ./template.yaml --capabilities CAPABILITY_IAM
```

Note that we needed to specify the `--capabilities` flag.
This indicates that we're acknowledging that the template will create an IAM Role as part of the deployment.

//TODO add cURL example to show the integration working.

Now, let's take a closer look at the `ResponseTemplates` section of the integration.
API Gateway uses [Velocity Templating Language](http://people.apache.org/~henning/velocity/html/ch02s02.html) to transform requests/responses according to pre-defined logic.
The service actually uses a subset of VTL (for some reason I'm unaware of).

In our example, let's suppose that we don't want to directly return the response from DynamoDB (like what the previous example does) and instead we want to add the list of tables to a custom field named `Tables`.
Also, the DynamoDB ListTables API call accepts two optional parameters, `ExclusiveStartTableName` and `Limit`.
Both of these parameters are used to support pagination.
Let's suppose that we don't want the default `Limit` of 100, but instead want to return only 10 table names at a time.

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Resources:
    ExampleRestApi:
        Type: AWS::ApiGateway::RestApi
        Properties:
            Name: MyRESTAPI

    FunctionsPath:
          Type: 'AWS::ApiGateway::Resource'
          Properties:
            RestApiId: !Ref ExampleRestApi
            ParentId: !GetAtt ExampleRestApi.RootResourceId
            PathPart: functions

    ListFunctionsGetMethod:
        Type: AWS::ApiGateway::Method
        Properties:
            OperationName: "ListFunctions"
            HttpMethod: GET
            ResourceId: !Ref FunctionsPath
            RestApiId: !Ref ExampleRestApi
            AuthorizationType: NONE
            Integration:
                Type: MOCK
                IntegrationResponses:
                  - ResponseTemplates:
                        application/json: "{\"hello\": \"Ferris\"}"
                    SelectionPattern: '2\d{2}'
                    StatusCode: 200
                RequestTemplates:
                    application/json: "{\n \"statusCode\": 200\n}"
            MethodResponses:
              - StatusCode: 200
            

    TablesPath:
          Type: 'AWS::ApiGateway::Resource'
          Properties:
            RestApiId: !Ref ExampleRestApi
            ParentId: !GetAtt ExampleRestApi.RootResourceId
            PathPart: tables

    ListTablesGetMethod:
        Type: AWS::ApiGateway::Method
        Properties:
            OperationName: "ListTables"
            HttpMethod: GET
            ResourceId: !Ref TablesPath
            RestApiId: !Ref ExampleRestApi
            AuthorizationType: NONE
            Integration:
              Type: AWS
              IntegrationHttpMethod: POST
              IntegrationResponses:
              - StatusCode: 200
                ResponseTemplates:
                  application/json: |
                    #set($inputRoot = $input.path('$'))
                    {
                      "Tables": [
                        #foreach($elem in $inputRoot.TableNames)
                          "$elem"
                        #if($foreach.hasNext),#end
                        #end
                      ]
                    }
              RequestTemplates:
                # This section tells API Gateway how to transform requests that are made to API Gateway when the content-type is set to application/json
                application/json: |
                  {"Limit":10}
              PassthroughBehavior: WHEN_NO_TEMPLATES
              RequestParameters:
                integration.request.header.X-Amz-Target: "'DynamoDB_20120810.ListTables'"
                integration.request.header.Content-Type: "'application/x-amz-json-1.0'"
              Uri: !Sub "arn:aws:apigateway:${AWS::Region}:dynamodb:path//"
              Credentials: !GetAtt ListTablesRole.Arn
            MethodResponses:
              - StatusCode: 200

    Deployment02:
      DependsOn:
        - ListTablesGetMethod
        - ListFunctionsGetMethod
      Type: 'AWS::ApiGateway::Deployment'
      Properties:
        RestApiId: !Ref ExampleRestApi
        Description: My deployment
        StageName: Prod

    ListTablesRole:
      Type: 'AWS::IAM::Role'
      Description: |
        IAM Role needed by API Gateway in order to have the correct permissions needed to call other AWS APIs on your behalf.
        See more info at https://docs.aws.amazon.com/apigateway/latest/developerguide/permissions.html
      Properties:
        AssumeRolePolicyDocument:
          Version: "2012-10-17"
          Statement:
            - Effect: Allow
              Principal:
                Service:
                  - apigateway.amazonaws.com
              Action:
                - 'sts:AssumeRole'
        Path: /
        Policies:
          - PolicyName: root
            PolicyDocument:
              Version: "2012-10-17"
              Statement:
                - Effect: Allow
                  Action: 'dynamodb:ListTables'
                  Resource: '*'

Outputs:
  HelloWorldApi:
    Description: "API Gateway endpoint URL for Prod stage"
    Value: !Sub "https://${ExampleRestApi}.execute-api.${AWS::Region}.amazonaws.com/Prod/"
```

Now let's take a moment to test this out.
First, we'll get the url for the API from the CloudFormation output.

```bash
aws cloudformation describe-stacks --stack-name my-artisinal-api --query "Stacks[*].Outputs[?OutputKey=='HelloWorldApi'].OutputValue" --output text
```

Now we'll use the response from that api call.
For me the response was `https://u568dumh9k.execute-api.us-west-2.amazonaws.com/Prod/`, yours will be different.

```bash
curl https://u568dumh9k.execute-api.us-west-2.amazonaws.com/Prod/tables
```

return
```text
{
  "Tables": [
          "my-example-table"
          ]
}
```

Remember when I said the `requestTemplates` maps content-types to a transformation?
Since we didn't specify a content-type, API Gateway will use [a default value](https://docs.aws.amazon.com/apigateway/latest/developerguide/request-response-data-mappings.html) of `application/json`.

> When the Content-Type header is absent in the request, API Gateway assumes that its default value is application/json

Let's see what happens when we specify a different value.

```bash
curl -H "Content-Type: text/html" https://u568dumh9k.execute-api.us-west-2.amazonaws.com/Prod/tables
```

Returns the following

```text
{"message": "Unsupported Media Type"}
```

This is because we told API Gateway to only pass requests through unmodified `WHEN_NO_TEMPLATES` which is "if there are no templates defined".
Since we defined at least one template, this flag tells API Gateway to reject all explicitly supported content types.
This is all super boring, but I describe it in much more detail in another [old blog post](https://rboyd.dev/089999bf-b973-42ed-9796-6167539269b8).