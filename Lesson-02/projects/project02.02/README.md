# Project 02.02 - Adding IAM Permissions

This project will pick up where the previous project left off.
We finished Project 1 with a Lambda function that will return a hardcoded response for any request.
Now we want to treat `GET` requests to the `/functions` and `/tables` paths separately.

Starting from our previous template, there are a couple ways we could probably do this.

The first approach is to define separate functions for `/functions`, `/tables`, and everything else .
This approach will allow us to scope the permissions for each of those functions so that the function associated with `/functions` only has permission to list the Lambda functions and `/tables` only has permission to read the DynamoDB tables.

The second appraoch is to slap all the logic for determining which path was called with which method into a single function and re-use the existing function that we have.
We're going to do both but we'll start with the former, then do it again with the latter.

We finished off project 1 with the following template, first we're going to remove all the comments and the extra template outputs to make the template a bit more concise.

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Transform: AWS::Serverless-2016-10-31
Description: >
  🗑🔥

  Sample SAM Template for 🗑🔥

# More info about Globals: https://github.com/awslabs/serverless-application-model/blob/master/docs/globals.rst
Globals:
  Function:
    Timeout: 3

Resources:
  HelloWorldFunction:
    Type: AWS::Serverless::Function
    Metadata:
        BuildMethod: makefile
    Properties:
      Handler: app.lambda_handler # this value no longer matters, but it needs to be present because APIs are hard
      Runtime: provided.al2
      CodeUri: .
      Events:
        HelloWorld:
          Type: Api # More info about API Event Source: https://github.com/awslabs/serverless-application-model/blob/master/versions/2016-10-31.md#api
          Properties:
            Path: /hello
            Method: get
Outputs:
  # ServerlessRestApi is an implicit API created out of Events key under Serverless::Function
  # Find out more about other implicit resources you can reference within SAM
  # https://github.com/awslabs/serverless-application-model/blob/master/docs/internals/generated_resources.rst#api
  HelloWorldApi:
    Description: "API Gateway endpoint URL for Prod stage for Hello World function"
    Value: !Sub "https://${ServerlessRestApi}.execute-api.${AWS::Region}.amazonaws.com/Prod/hello/"
  HelloWorldFunction:
    Description: "Hello World Lambda Function ARN"
    Value: !GetAtt HelloWorldFunction.Arn
  HelloWorldFunctionIamRole:
    Description: "Implicit IAM Role created for Hello World function"
    Value: !GetAtt HelloWorldFunctionRole.Arn
```

After pruning out the parts that aren't relevant to this exercise we're left with something a bit earier to work with.

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Transform: AWS::Serverless-2016-10-31

Globals:
  Function:
    Timeout: 3

Resources:
  HelloWorldFunction:
    Type: AWS::Serverless::Function
    Metadata:
        BuildMethod: makefile
    Properties:
      Handler: app.lambda_handler
      Runtime: provided.al2
      CodeUri: .
      Events:
        HelloWorld:
          Type: Api
          Properties:
            Path: /hello
            Method: get
Outputs:
  HelloWorldApi:
    Description: "API Gateway endpoint URL for Prod stage for Hello World function"
    Value: !Sub "https://${ServerlessRestApi}.execute-api.${AWS::Region}.amazonaws.com/Prod/hello/"
```

Okay, now that that's out of our way, let's start writing some Rust.
Create the following folder structure by running the following commands from the root of your project (where the `template.yaml` file is located):

```bash
cargo new functions
cargo new tables
cargo new other
```

```
🗑🔥
│   README.md
│   samconfig.toml
│   template.yaml
│
└───functions
│   │  Cargo.toml
│   └──src
│       │  main.rs
|   
└───tables
│   │  Cargo.toml
│   └──src
│       │  main.rs
|
└───other
    │  Cargo.toml
    └──src
        │  main.rs
```

We still have the `src/` folder, `Cargo.toml`, and `Makefile` files at the root of our project.
Just for the sake of brevity, let's copy the `main.rs` file from `./src/main.rs` to each of those three new crates we just made.
We'll update each individually, but at least this will give us a headstart on the Rust implmentation.

Once you've done that, let's update our CloudFormation template to reflect the new functions we've created.

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Transform: AWS::Serverless-2016-10-31

Globals:
  Function:
    Timeout: 3

Resources:
  ListFunctionsFunction:
    Type: AWS::Serverless::Function
    Metadata:
        BuildMethod: makefile
    Properties:
      Handler: app.lambda_handler
      Runtime: provided.al2
      CodeUri: ./functions
      Events:
        HelloWorld:
          Type: Api
          Properties:
            Path: /functions
            Method: get
  ListTablesFunction:
    Type: AWS::Serverless::Function
    Metadata:
        BuildMethod: makefile
    Properties:
      Handler: app.lambda_handler
      Runtime: provided.al2
      CodeUri: ./tables
      Events:
        HelloWorld:
          Type: Api
          Properties:
            Path: /tables
            Method: get
  EverythingElseFunction:
    Type: AWS::Serverless::Function
    Metadata:
        BuildMethod: makefile
    Properties:
      Handler: app.lambda_handler
      Runtime: provided.al2
      CodeUri: ./other
      Events:
        HelloWorld:
          Type: Api
          Properties:
            Path: /{proxy+}
            Method: any
Outputs:
  HelloWorldApi:
    Description: "API Gateway endpoint URL for Prod stage for Hello World function"
    Value: !Sub "https://${ServerlessRestApi}.execute-api.${AWS::Region}.amazonaws.com/Prod/hello/"
```

Now, let's walk through this one resource at a time to explain what's happening.

```yaml
  ListFunctionsFunction:
    Type: AWS::Serverless::Function
    Metadata:
        BuildMethod: makefile
    Properties:
      Handler: app.lambda_handler
      Runtime: provided.al2
      CodeUri: ./functions
      Events:
        HelloWorld:
          Type: Api
          Properties:
            Path: /functions
            Method: get
```

We've renamed the function from 'HelloWorldFunction' to something a bit more descriptive.
This name is called the "Logical Resource ID" and is used to refer to this resource elsewhere in the template.
We've moved our Rust logic from the root of the application's folder to its own directory named `/functions`.
This is the folder that was created when we ran `cargo new functions`.
We're using the `Makefile` approach as we did previously, but we also need to copy the existing `Makefile` into the `./functions` folder.

We need to slightly modify the `Makefile` for it to work with this new function.
The SAM CLI expects the Make definition to be named `build-[logical resource id]` so because we renamed our function from `HelloWorldFunction` to `ListFunctionsFunction`, we need to update the `Makefile` to reflect this.

```bash
build-ListFunctionsFunction:
	cargo build --release
	cp ./target/release/bootstrap $(ARTIFACTS_DIR)
```

Next we will update the `Events` seciton of the function's definition.
Previously, we mapped the `GET` requests for path `/hello` to our function, but we need to change this to the `/functions` path.

We repeate these steps for the `ListTablesFunction` resource and the `EverythingElseFunction` resource.
The `ListTablesFunction` is unremarkable from the `ListFunctionsFunction` resource so I will trust that you can update that one without additional instructions.
The `EverythingElseFunction` is unique in that the path portion is `/{proxy+}` instead of `/tables` or `/functions`.
the curly brackets tell API Gateway to 'capture' whatever value was placed in that location in the path and relay it to the Lambda function (assigned to a variable named `proxy`) when the funtion is invoked.
Project 3 will dive into how that capture and relay process looks to users.
The `+` indicates that the capture is greedy.
instead of trying to explain what this means through some formal definition I will just show you some examples and let your lizard brain pattern matching skills take over.

Suppose we have the following API definition.

- `/foo`
- `/foo/{baz}`
- `/bar`
- `/bar/{buzz+}`
- `/{owner+}`

With some sample inputs:

- A request to `/foo` will directly match the first entry
- A request to `/foo/richard` will match the second entry and assign `baz=richard`
- A request to `/foo/richard/something` will not match because `baz` is not 'greedy'
- A request to `/bar/jeff` will match with the 4 entry and assign `buzz=jeff`
- A request to `/bar/jeff/something` will match the 4th entry and assign `buzz=jeff/something` because `buzz` is greedy
- A request to `/some/random/entry` will match the last entry and assign `owner=/some/random/entry` because `owner` is greedy <sup>This is now a socialism guide</sup>.

Next we need to grant the individual Lambda functions the IAM permissions they will need to retrieve the list of functions or tables.
For the `/functions` endpoint, we need to grant it the [`ListFunctions` permission](https://docs.aws.amazon.com/service-authorization/latest/reference/list_awslambda.html).
For the `/tables` endpoint, we need to grant it the [`ListTables` permission](https://docs.aws.amazon.com/service-authorization/latest/reference/list_amazondynamodb.html).

We assign these permissions by adding an IAM policy document to each function's definition.
Your updated `template.yaml` should look like this now:

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Transform: AWS::Serverless-2016-10-31

Globals:
  Function:
    Timeout: 3

Resources:
  ListFunctionsFunction:
    Type: AWS::Serverless::Function
    Metadata:
        BuildMethod: makefile
    Properties:
      Handler: app.lambda_handler
      Runtime: provided.al2
      CodeUri: ./functions
      Events:
        HelloWorld:
          Type: Api
          Properties:
            Path: /functions
            Method: get
      Policies:
      - Statement:
        - Sid: ListFunctionsPolicy
          Effect: Allow
          Action:
          - lambda:ListFunctions
          Resource: '*'
  ListTablesFunction:
    Type: AWS::Serverless::Function
    Metadata:
        BuildMethod: makefile
    Properties:
      Handler: app.lambda_handler
      Runtime: provided.al2
      CodeUri: ./tables
      Events:
        HelloWorld:
          Type: Api
          Properties:
            Path: /tables
            Method: get
      Policies:
      - Statement:
        - Sid: ListTablessPolicy
          Effect: Allow
          Action:
          - dynamodb:ListTables
          Resource: '*'
  EverythingElseFunction:
    Type: AWS::Serverless::Function
    Metadata:
        BuildMethod: makefile
    Properties:
      Handler: app.lambda_handler
      Runtime: provided.al2
      CodeUri: ./other
      Events:
        HelloWorld:
          Type: Api
          Properties:
            Path: /{proxy+}
            Method: any
Outputs:
  HelloWorldApi:
    Description: "API Gateway endpoint URL for Prod stage for Hello World function"
    Value: !Sub "https://${ServerlessRestApi}.execute-api.${AWS::Region}.amazonaws.com/Prod/hello/"
```

With these changes, we've instructed CloudFormation that we want each function to have additional permissions that are needed for the function to do what our API needs it to do.
Now let's update the function code to perform this new logic.

### ListFunctionsFunction
Here's an [example](https://github.com/awslabs/aws-sdk-rust/blob/main/sdk/examples/lambda-list-functions/src/main.rs), go wild!

### ListTablesFunction
Here's an [example](https://github.com/awslabs/aws-sdk-rust/blob/main/sdk/examples/dynamo-list-tables/src/main.rs), go wild!

### EverythingElseFunction
This function doesn't have any additional permissions and doesn't need to use the AWS SDK for Rust, so it should be similar to Project 1
