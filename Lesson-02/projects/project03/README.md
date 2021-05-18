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

    Method:
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
```

Let's deploy this and make sure it works.

```text
aws cloudformation deploy --template-file ./template.yaml --stack-name my-artisinal-api
```