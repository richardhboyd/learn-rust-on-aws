# Project 02.01 - Just the Basics

Create a SAM application that consists of an API Gateway REST API that proxies all requests to a single Lambda function.
This function should return the following json response `{"hello": "world"}`

We'll start with an AWS Cloud9 AL2 environment.

The Serverless Application Model CLI should already be installed and configured by default.
We can check this by running `sam --version` in the terminal.
If you get an error, yell at Timir Karia.

We'll need to install Rust before we can use it.

```bash
curl https://sh.rustup.rs -sSf | sh -s -- -y --default-toolchain nightly
source $HOME/.cargo/env`
```

Because the tech industry can't standardize on if a CLI should create a new folder in the current directory and initialize the new project in that folder or if the CLI should initialize the new project in the current directory, we're going to YOLO it and jsut run the `init` command from the root of our environment.

```text
Feder08:~/environment/learn-rust-on-aws (main) $ sam init

        SAM CLI now collects telemetry to better understand customer needs.

        You can OPT OUT and disable telemetry collection by setting the
        environment variable SAM_CLI_TELEMETRY=0 in your shell.
        Thanks for your help!

        Learn More: https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/serverless-sam-telemetry.html

Which template source would you like to use?
        1 - AWS Quick Start Templates
        2 - Custom Template Location
```

Pick option 1.
If the options have changed, yell at Jacob Fuss.

```text
What package type would you like to use?
        1 - Zip (artifact is a zip uploaded to S3)
        2 - Image (artifact is an image uploaded to an ECR image repository)
Package type: 
```

As before, pick option 1 to indicate that we will be packaging our Lambda function via ZIP instead of the new Docker Image format. 
This is a great time to point out that SAM treats Lambda functions as the center of literally every serverless project.
Are you creating an event-driven application that doesn't need functions and instead uses API Gateway and EventBridge?
Too bad, you need to answer this very specific implentation question about Lambda functions before you can proceed on your quest.

```text
Which runtime would you like to use?
        1 - nodejs14.x
        2 - python3.8
        3 - ruby2.7
        4 - go1.x
        5 - java11
        6 - dotnetcore3.1
        7 - nodejs12.x
        8 - nodejs10.x
        9 - python3.7
        10 - python3.6
        11 - python2.7
        12 - ruby2.5
        13 - java8.al2
        14 - java8
        15 - dotnetcore2.1
Runtime: 
```

Find Rust on the list and pick that number.
Just kidding, Lambda only provides managed runtimes for LTS languages, we've got 4 versions of Python and 3 versions each of nodeJS and Java, but Rust and .NET 5 developers don't deserve to be loved.
We'll pick `9` for `python3.7` and we'll fix it later.

```text
Project name [sam-app]: 🗑🔥
```

I'm going to name the project `🗑🔥`, you can name yours whatever you want.

```text
AWS quick start application templates:
        1 - Hello World Example
        2 - EventBridge Hello World
        3 - EventBridge App from scratch (100+ Event Schemas)
        4 - Step Functions Sample App (Stock Trader)
Template selection: 
```

Pick `1` for `Hello World Example`, we'll cover the other options in a later lesson.

```text
Template selection: 1

    -----------------------
    Generating application:
    -----------------------
    Name: 🗑🔥
    Runtime: python3.7
    Dependency Manager: pip
    Application Template: hello-world
    Output Directory: .
    
    Next steps can be found in the README file at ./🗑🔥/README.md
        

SAM CLI update available (1.23.0); (1.19.0 installed)
To download: https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/serverless-sam-cli-install.html
```

Cool, now the SAM CLI has scaffolded an example serverless application in a folder named 🗑🔥.

let's deploy our new application.

```bash
cd 🗑🔥
sam deploy --guided
```

The SAM CLI will prompt you for deployment options, we will walk through each of these in turn.

```text
Configuring SAM deploy
======================

        Looking for config file [samconfig.toml] :  Not found

        Setting default arguments for 'sam deploy'
        =========================================
        Stack Name [sam-app]: U1F5D1-U1F525
        AWS Region [us-west-2]: 
        #Shows you resources changes to be deployed and require a 'Y' to initiate deploy
        Confirm changes before deploy [y/N]: N
        #SAM needs permission to be able to create roles to connect to the resources in your template
        Allow SAM CLI IAM role creation [Y/n]: Y
        HelloWorldFunction may not have authorization defined, Is this okay? [y/N]: y
        Save arguments to configuration file [Y/n]: Y
        SAM configuration file [samconfig.toml]: 
        SAM configuration environment [default]: 

        Looking for resources needed for deployment: Not found.
        Creating the required resources...
        Successfully created!

                Managed S3 bucket: aws-sam-cli-managed-default-samclisourcebucket-1i0sbjhna2lzd
                A different default S3 bucket can be set in samconfig.toml

        Saved arguments to config file
        Running 'sam deploy' for future deployments will use the parameters saved above.
        The above parameters can be changed by modifying samconfig.toml
        Learn more about samconfig.toml syntax at 
        https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/serverless-sam-cli-config.html

Uploading to U1F5D1-U1F525/c6ce8fa8b5a97dd022ecd006536eb5a4  847 / 847  (100.00%)
```
The Options presented are as follows:

```text
Stack Name [sam-app]: U1F5D1-U1F525
```

AWS Cloudformation is a bit more strict than the SAM CLI, so we need to come up with an alphanumeric name for the stack name. (//TODO Describe what a stack is).
I'm going to name mine `U1F5D1-U1F525`, you can name yours whatever you would like.
The default `sam-app` is also an appropriate response.
The general pattern for these prompts is that an answer by itself in `[]` is the default option.
If multiple options are presented together, the capitalized option is the default (such as `[y/N]` where `N` or `No` is the default)

```text
AWS Region [us-west-2]: us-west-2
```

In an ideal world, this will be the region where your Cloud9 environment is deployed,
You can change it to another region if you would like to learn a very painful lesson in "why does the console only show a single region at a time?" this early in your cloud journey.
Again, the default should be fine.

```text
Confirm changes before deploy [y/N]: N
```

This is asking you if you want to be asked to deploy your infrastructure.
Anyone who replies with a `Y` is a cop.

```text
 Allow SAM CLI IAM role creation [Y/n]: Y
 ```
 
Even a very simple 'hello, world' Lambda function needs AWS IAM permissions to operate, this prompt is asking you if you're cool with IAM Roles being created for you.
There are cases where this is important, what we are doing today is not one of those so we'll accept thedefault and YOLO the role creation.

```text
HelloWorldFunction may not have authorization defined, Is this okay? [y/N]: y
```

We'll cover what `HelloWorldFunction` is in just a moment.
I *think* this prompt is referring to authorization to invoke the function?
If we accept the default, the deployment won't work, so let's pick `y` and figure out what that means later.

```text
Save arguments to configuration file [Y/n]: Y
```

As we fill out this personality test of a questionaire to deploy our appication, we can tell the SAM CLIto save our responses so we don't have to play 20-questions for every iteration of our code. The defaultwill save us a bunch of time.

```text
SAM configuration file [samconfig.toml]: samconfig.toml
```

Since we told the CLI that it's cool to save our configuration, it will now need to know where exactlyto save that configuration.
If you've noticed the pattern, you know what I'm about to say.
The default is fine for this option.

```text
SAM configuration environment [default]: 
```

This option allows you to specify several deployment targets, such as deployment pipeline stages.
Since we're not building a CI/CD system or creating multiple deployment targets today, the default is a fine choice.

Now you're going to see a whole bunch of text fly past you.
Treat this like a wood chipper, ignore it until it stops moving.

```text
Uploading to U1F5D1-U1F525/c6ce8fa8b5a97dd022ecd006536eb5a4  847 / 847  (100.00%)

        Deploying with following values
        ===============================
        Stack name                   : U1F5D1-U1F525
        Region                       : us-west-2
        Confirm changeset            : False
        Deployment s3 bucket         : aws-sam-cli-managed-default-samclisourcebucket-1i0sbjhna2lzd
        Capabilities                 : ["CAPABILITY_IAM"]
        Parameter overrides          : {}
        Signing Profiles             : {}

Initiating deployment
=====================
HelloWorldFunction may not have authorization defined.
Uploading to U1F5D1-U1F525/ff03fba2bd37c2ae7a86a4f43b67a729.template  1118 / 1118  (100.00%)

Waiting for changeset to be created..

CloudFormation stack changeset
-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
Operation                                         LogicalResourceId                                 ResourceType                                      Replacement                                     
-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
+ Add                                             HelloWorldFunctionHelloWorldPermissionProd        AWS::Lambda::Permission                           N/A                                             
+ Add                                             HelloWorldFunctionRole                            AWS::IAM::Role                                    N/A                                             
+ Add                                             HelloWorldFunction                                AWS::Lambda::Function                             N/A                                             
+ Add                                             ServerlessRestApiDeployment47fc2d5f9d             AWS::ApiGateway::Deployment                       N/A                                             
+ Add                                             ServerlessRestApiProdStage                        AWS::ApiGateway::Stage                            N/A                                             
+ Add                                             ServerlessRestApi                                 AWS::ApiGateway::RestApi                          N/A                                             
-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

Changeset created successfully. arn:aws:cloudformation:us-west-2:072326518754:changeSet/samcli-deploy1621131225/e06e615e-d919-456b-957c-96e94d66349f


2021-05-16 02:13:55 - Waiting for stack create/update to complete

CloudFormation events from changeset
-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
ResourceStatus                                    ResourceType                                      LogicalResourceId                                 ResourceStatusReason                            
-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
CREATE_IN_PROGRESS                                AWS::IAM::Role                                    HelloWorldFunctionRole                            -                                               
CREATE_IN_PROGRESS                                AWS::IAM::Role                                    HelloWorldFunctionRole                            Resource creation Initiated                     
CREATE_COMPLETE                                   AWS::IAM::Role                                    HelloWorldFunctionRole                            -                                               
CREATE_IN_PROGRESS                                AWS::Lambda::Function                             HelloWorldFunction                                -                                               
CREATE_COMPLETE                                   AWS::Lambda::Function                             HelloWorldFunction                                -                                               
CREATE_IN_PROGRESS                                AWS::Lambda::Function                             HelloWorldFunction                                Resource creation Initiated                     
CREATE_IN_PROGRESS                                AWS::ApiGateway::RestApi                          ServerlessRestApi                                 -                                               
CREATE_IN_PROGRESS                                AWS::ApiGateway::RestApi                          ServerlessRestApi                                 Resource creation Initiated                     
CREATE_COMPLETE                                   AWS::ApiGateway::RestApi                          ServerlessRestApi                                 -                                               
CREATE_IN_PROGRESS                                AWS::Lambda::Permission                           HelloWorldFunctionHelloWorldPermissionProd        Resource creation Initiated                     
CREATE_IN_PROGRESS                                AWS::Lambda::Permission                           HelloWorldFunctionHelloWorldPermissionProd        -                                               
CREATE_IN_PROGRESS                                AWS::ApiGateway::Deployment                       ServerlessRestApiDeployment47fc2d5f9d             -                                               
CREATE_IN_PROGRESS                                AWS::ApiGateway::Deployment                       ServerlessRestApiDeployment47fc2d5f9d             Resource creation Initiated                     
CREATE_COMPLETE                                   AWS::ApiGateway::Deployment                       ServerlessRestApiDeployment47fc2d5f9d             -                                               
CREATE_IN_PROGRESS                                AWS::ApiGateway::Stage                            ServerlessRestApiProdStage                        -                                               
CREATE_IN_PROGRESS                                AWS::ApiGateway::Stage                            ServerlessRestApiProdStage                        Resource creation Initiated                     
CREATE_COMPLETE                                   AWS::ApiGateway::Stage                            ServerlessRestApiProdStage                        -                                               
CREATE_COMPLETE                                   AWS::Lambda::Permission                           HelloWorldFunctionHelloWorldPermissionProd        -                                               
CREATE_COMPLETE                                   AWS::CloudFormation::Stack                        U1F5D1-U1F525                                     -                                               
-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

CloudFormation outputs from deployed stack
------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
Outputs                                                                                                                                                                                              
------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
Key                 HelloWorldFunctionIamRole                                                                                                                                                        
Description         Implicit IAM Role created for Hello World function                                                                                                                               
Value               arn:aws:iam::072326518754:role/U1F5D1-U1F525-HelloWorldFunctionRole-1P8YM9BJFLHWX                                                                                                

Key                 HelloWorldApi                                                                                                                                                                    
Description         API Gateway endpoint URL for Prod stage for Hello World function                                                                                                                 
Value               https://t4yqgjv9y6.execute-api.us-west-2.amazonaws.com/Prod/hello/                                                                                                               

Key                 HelloWorldFunction                                                                                                                                                               
Description         Hello World Lambda Function ARN                                                                                                                                                  
Value               arn:aws:lambda:us-west-2:072326518754:function:U1F5D1-U1F525-HelloWorldFunction-P35XTFPY8e1H                                                                                     
------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

Successfully created/updated stack - U1F5D1-U1F525 in us-west-2
```

> Note: If there are odd discrepencies in the timestamps or randomly generated file names, you can safely ignore them. It's usually because I wrote this guide in several sittings and I'm not too worried about those details just yet.

Now that the CLI has calmed down, examine the final section named `Outputs`.
In my repsonse it shows a key/value pair with a key named `HelloWorldApi` with a value of `https://t4yqgjv9y6.execute-api.us-west-2.amazonaws.com/Prod/hello/`.
Yours *WILL* be different.
If yours is the same, please yell at Alan Tan.
This is the url for the API we just delpoyed.
I know youre probably clammoring to do something cool so we'll poke this API a bit, then we'll dive into explaining what happened.

You can invoke the API with `cUrl`:

```bash
curl https://t4yqgjv9y6.execute-api.us-west-2.amazonaws.com/Prod/hello
```

```text
Feder08:~/environment/🗑🔥 $ curl https://t4yqgjv9y6.execute-api.us-west-2.amazonaws.com/Prod/hello/
{"message": "hello world"}Feder08:~/environment/🗑🔥 $ 
```

Whitespace is very hard, but we see that the API returned a JSON object `{"message": "hello world"}`.
This is very close to what the project statement asks for, but it's not quite right.
let's also check other paths by replacing the `/hello/` path with `/goodbye`

```bash
curl https://t4yqgjv9y6.execute-api.us-west-2.amazonaws.com/Prod/goodbye
```

```text
Feder08:~/environment/🗑🔥 $ curl https://t4yqgjv9y6.execute-api.us-west-2.amazonaws.com/Prod/goodbye/
{"message":"Missing Authentication Token"}Feder08:~/environment/🗑🔥 $
```

This is confusing because it's a JSON response with a `"message"` entry, but this one didn't come from our Lambda function.
This is the error you will receive if you try to make a request to an API Gateway REST API path that isn't defined.
Our project statement wants us to catch any possible path and route it to the same Lambda function.
So, we have two tasks ahead of us:

- adjust the response to return `{"hello": "world"}`
- change the API configuration to accept any possible path
- (off by one errors are already showing up) convert the Lambda function to Rust instead of Python3.7

We'll start with the conversion to Rust

## Adding Rust

Inside the 🗑🔥 directory we have the following files/folders

```
🗑🔥
│   __init__.py
│   README.md
│   samconfig.toml
│   template.yaml
│
└───events
│   |   event.json
│   
└───hello_world
│   |   __init__.py
│   |   app.py
│   |   requiremetns.txt
│   
└───tests
    │   __init__.py
    │
    └───unit
        │   __init__.py
        │   test_harness.py
```

Delete the following Python related files/folders:

```bash
rm __init__.py
rm -rf ./hello_world
rm -rf ./tests
```

Now we've broken the application, fixing it is left as an exercise for the reader.

----

Just kidding, I wouldn't do that to you.

Let's go ahead and make this application a bit rusty.
From the root of the project folder (🗑🔥) create the `Cargo.toml` file and the `src/` folder.
Usually we would just use `cargo new ...` but we already have the folder created and I don't know how to create a create in the current directory.

```bash
touch Cargo.toml
mkdir src
touch src/main.rs
```

Let's walk through the `Cargo.toml` file first.

```toml
[package]
name = "hello-world-lambda-function"
version = "0.1.0"
edition = "2018"
```

You know this part, it's the usual `[package]` information

```toml
[dependencies]
tokio = { version = "1", features = ["full"] }
serde = { version = "1.0.82", features = ["derive"] }
serde_json = { version = "1.0.33", features = ["raw_value"] }

lambda_runtime = "0.3"
```
We're going to include the [Rust Runtime for AWS Lambda](https://github.com/awslabs/aws-lambda-rust-runtime).
This runtime has convenience helpers that will do some Rust stuff I don't fully understand.

```toml
[[bin]]
name = "bootstrap"
path = "src/main.rs"
```

Finally, we need to complete our ceremony by packaging our Rust binary in a very specific way.
If you don't get this incantation just right, it will not work and it will be all your fault.
There's no way we could possibly make this easier.

Because Rust is not an Officially Supported Runtime™ by the AWS Lambda service, we have to use a mechanism known as [~~just use a real language instead~~ Custom Runtimes](https://docs.aws.amazon.com/lambda/latest/dg/runtimes-custom.html).
The short version of that doc page is that you must create a file named `bootstrap` that is executable that will be executed to invoke the logic of your function.
In officially supported runtimes, such as Python, the Lambda service will know how to invoke the logic of your Lambda function.
In custom runtimes, we have to tell the Lambda service how to do this.
The contract is that we will put these instructions in a file named `bootstrap` and Lambda will execute it when the function is invoked.

We'll use Cargo's ability to specify a binary name to avoid having to manually rename the file.
If you're not using Cloud9, you're about to get a crash course in cross-compiling.
The custom runtime environment uses Amazon Linux 2 and we're, hopefully, also using AL2 in Cloud9 so there shouldn't be a need to compile for the Lambda OS, but if you're on a Mac or some other linux variant ... good luck.
The original Rust Runtime for AWS Lambda [blog post](https://aws.amazon.com/blogs/opensource/rust-runtime-for-aws-lambda/) has a good overview of the steps needed.

Now we'll get to the Rust code:

```rust
use lambda_runtime::{handler_fn, Context};
use serde::{Deserialize, Serialize};

pub type Error = Box<dyn std::error::Error + Send + Sync + 'static>;

#[derive(Deserialize)]
struct Request {
    path: String,
}

#[derive(Serialize)]
struct Response {
    body: ResponseBody,
    // This should be a u32, but API Gateway actually expects a String that looking like an int for some reason.
    statusCode: String
}

#[derive(Serialize)]
struct ResponseBody {
    hello: String
}

#[tokio::main]
async fn main() -> Result<(), Error> {
    let func = handler_fn(my_handler);
    lambda_runtime::run(func).await?;
    Ok(())
}

pub(crate) async fn my_handler(event: Request, ctx: Context) -> Result<Response, Error> {

    let resp = Response {
        body: ResponseBody {hello: String::from("world")},
        statusCode: String::from("200")
    };

    Ok(resp)
}
```
Next we will tell the SAM CLI how to build our application.
Create a `Makefile`

`touch Makefile`

Add the following contents.
This will tell SAM how to build our application and place it in the right directory for deployment.

```text
build-HelloWorldFunction:
	cargo build --release
	cp ./target/release/bootstrap $(ARTIFACTS_DIR)
```

> Note: those are tabs and not spaces

Our last change is to update our CloudFormation template to reflect our new runtime.
Before we start making changes, your template should look something like this.

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
    Type: AWS::Serverless::Function # More info about Function Resource: https://github.com/awslabs/serverless-application-model/blob/master/versions/2016-10-31.md#awsserverlessfunction
    Properties:
      CodeUri: hello_world/
      Handler: app.lambda_handler
      Runtime: python3.7
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

Update the template with these values.
The options that will need to be changed are:
- `CodeUri`
- `Runtime`
- `Metadata`

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

We should now be able to build this application and deploy it.
Running `sam build` will build the Rust application and prepare it for packaging and deployment.
Running `sam deploy` will deploy it with the options we used last time, by reading the options from the `samconfig.toml` file.

> Note: this will deploy but attempting to invoke it will fail with `{"message": "Internal server error"}` because the response from our function is not quite right.
We are returning a literal string `{"hello":"world"}` but what we actually need to return is something like `{"body": "{\"hello\":\"world\"}","statusCode": 200}`.
I can't figure out how to do this without writing a boat load of Serialization logic so hopefully someone else knows some Rust tricks.

//TODO catch all the API Gateway paths by using the `{proxy+}` route
