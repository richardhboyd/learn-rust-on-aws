# Project 03.02 - CDK Cont'd

We'll start by making sure Rust is installed and using the nightly toolchain.

```bash
curl https://sh.rustup.rs -sSf | sh -s -- -y --default-toolchain nightly
source $HOME/.cargo/env
```

I'll explain later in the guide why we're using the nightly toolchain.

Next we'll create a directory to hold our CDK application.
For the rest of today's project, any reference to 'the root of the project' are referring to `project02/`

```bash
mkdir project02
cd project02/
```

Now we'll create a scaffolded CDK application.
We're going to use Python as the language for defining our infrastructure for now (because CDK doesn't support defining your infrastructure in Rust yet).

```bash
cdk init app --language python
```

This will generate a lot of console text as it explains what it's doing.

```text
Applying project template app for python

# Welcome to your CDK Python project!

This is a blank project for Python development with CDK.

The `cdk.json` file tells the CDK Toolkit how to execute your app.

This project is set up like a standard Python project.  The initialization
process also creates a virtualenv within this project, stored under the `.venv`
directory.  To create the virtualenv it assumes that there is a `python3`
(or `python` for Windows) executable in your path with access to the `venv`
package. If for any reason the automatic creation of the virtualenv fails,
you can create the virtualenv manually.

To manually create a virtualenv on MacOS and Linux:


$ python3 -m venv .venv


After the init process completes and the virtualenv is created, you can use the following
step to activate your virtualenv.


$ source .venv/bin/activate


If you are a Windows platform, you would activate the virtualenv like this:


% .venv\Scripts\activate.bat


Once the virtualenv is activated, you can install the required dependencies.


$ pip install -r requirements.txt


At this point you can now synthesize the CloudFormation template for this code.


$ cdk synth


To add additional dependencies, for example other CDK libraries, just add
them to your `setup.py` file and rerun the `pip install -r requirements.txt`
command.

## Useful commands

 * `cdk ls`          list all stacks in the app
 * `cdk synth`       emits the synthesized CloudFormation template
 * `cdk deploy`      deploy this stack to your default AWS account/region
 * `cdk diff`        compare deployed stack with current state
 * `cdk docs`        open CDK documentation

Enjoy!

Initializing a new git repository...
Please run 'python3 -m venv .venv'!
Executing Creating virtualenv...
✅ All done!
```

Okay, now we have an empty project created.
Note the third to last line in the console output asks you to run `python3 -m venv .venv`.
This will create a python virtual environment in the directory `.venv` at the root of your project.

Let's go ahead and activate the virtual environment so our Python/CDK commands will use the local virtual environment

```bash
source .venv/bin/activate
```

If it works, you should see the prefix of the command line prompt change so that it starts with `(.venv)`

```bash
(.venv) Feder08:~/environment/project02 (master) $ 
```

First, we'll create our Lambda function code.

## Lambda Function Code

From the root of the project, run `cargo new my-example-function` to create a folder named `my-example-function` and replace the contents of the following files.

<details>
  <summary>Cargo.toml</summary>

Running this command form the root of the project will replace the contents of `Cargo.toml` with the appropriate dependencies.
```bash
cat << EOF > my-example-function/Cargo.toml
[package]
name = "my-example-function"
version = "0.1.0"
edition = "2021"

[dependencies]
dynamodb = { git = "https://github.com/awslabs/aws-sdk-rust", tag = "v0.0.12-alpha", package = "aws-sdk-dynamodb" }
aws-types = { git = "https://github.com/awslabs/aws-sdk-rust", tag = "v0.0.12-alpha", package = "aws-types" }

tokio = { version = "1", features = ["full"] }
serde = { version = "1.0.82", features = ["derive"] }
serde_json = { version = "1.0.33", features = ["raw_value"] }

lambda_runtime = "0.3"

[[bin]]
name = "bootstrap"
path = "src/main.rs"
EOF
```

</details>

<details>
  <summary>src/main.rs</summary>

```bash
cat << EOF > my-example-function/src/main.rs
use std::env;

use lambda_runtime::{handler_fn, Context};
use serde::{Deserialize, Serialize};

use dynamodb::{Client, Config, Region};
use aws_types::region::{EnvironmentProvider, ProvideRegion};


pub type Error = Box<dyn std::error::Error + Send + Sync + 'static>;

#[derive(Deserialize)]
struct APIRequest {
    body: String,
}

#[derive(Serialize, Deserialize)]
struct Request {
    command: String,
    author: String
}

#[derive(Serialize)]
#[serde(rename_all = "camelCase")]
struct Response {
    status_code: String,
    body: String,
}

#[tokio::main]
async fn main() -> Result<(), Error> {
    let func = handler_fn(my_handler);
    lambda_runtime::run(func).await?;
    Ok(())
}

pub(crate) async fn my_handler(event: APIRequest, _ctx: Context) -> Result<Response, Error> {
    let region = EnvironmentProvider::new()
        .region()
        .unwrap_or_else(|| Region::new("us-west-2"));
    let config = Config::builder().region(region).build();
    let client = Client::from_conf(config);

    let r: Request = serde_json::from_str(&event.body)?;
    let command = r.command;
    
    let table_name = env::var("TABLE_NAME").unwrap();

    //  This Lists the tables, change it to a dynamodb get_item request where the primary key and sort key are extracted from the request.
    match client.get_item().table_name(table_name).send().await {
        Ok(resp) => {
            let responses = resp.table_names.unwrap_or_default();
        }
        Err(e) => {
            println!("Got an error retrieving items:");
            println!("{}", e);
        }
    };
    // End dynamodb bits

    let resp = Response {
        status_code: "200".to_string(),
        body: format!("Command {} executed.\n", command),
    };

    Ok(resp)
}
EOF
```
</details>

Now we'll compile the binary with the following command (note: this is run from the root of the project):

`cargo +nightly build --release --out-dir=./my-example-function/out -Z unstable-options --manifest-path ./my-example-function/Cargo.toml`

This will place all of the build artifacts in the usual `target/release/` directory, but it will also copy just the binary to a directory named `out/`.
This will be important for the later CDK step.

## CDK Specifics

First, we will update our CDK application to take a dependency on the CDK module for AWS Lambda (in CDK V2, this won't be necessary, but we'll discuss that later)


add `aws-cdk.aws-lambda==X.YY.ZZ` to `setup.py` within `install_requires` section.

```python
    install_requires=[
        "aws-cdk.core==1.111.0",
        "aws-cdk.aws-lambda==1.111.0",
        "aws-cdk.aws-apigateway==1.111.0",
        "aws-cdk.aws-dynamodb==1.111.0"
    ],
```

Install the Python dependencies

```text
pip install -r ./requirements.txt
```

Update the `project02/project02_stack.py` file to create your AWS resources.

```python
import aws_cdk.core as cdk
import aws_cdk.aws_lambda as lambda_
import aws_cdk.aws_apigateway as apigw
import aws_cdk.aws_dynamodb as ddb

class Project02Stack(cdk.Stack):

    def __init__(self, scope: cdk.Construct, construct_id: str, **kwargs) -> None:
        super().__init__(scope, construct_id, **kwargs)
        
        table = ddb.Table(
            self, "MyTable",
            partition_key={ 'name': 'pk', 'type': ddb.AttributeType.STRING },
            sort_key={ 'name': 'sk', 'type': ddb.AttributeType.STRING }
        )

        function_reference = lambda_.Function(
            self, "Singleton",
            code=lambda_.Code.from_asset('./my-example-function/out'),
            handler="index.main",
            runtime=lambda_.Runtime.PROVIDED_AL2,
        )
        apigw.LambdaRestApi(self, "MyAPI", handler=function_reference)
        
        table.grant_read_write_data(function_reference)
        function_reference.add_environment(key="TABLE_NAME", value=table.table_name)
```

Note that the `code` parameter is `lambda_.Code.from_asset('./my-example-function/out')`, which means that we're telling CDK to grab the contents of a directory, zip it up, and use that as the function's 'code artifact'.
If we hadn't used the `out-dir` feature when we compiled our application, we would have had to either (a) point to the `target/release/` directory which would bring all of the build artifacts with it or (b) manually move the `bootstrap` executable to a new directory.
Using `out-dir` helps us avoid having to do that my telling Cargo to move the executable for us.

now we're ready to run `cdk synth` (to see the synthesized CloudFormation stack) or `cdk deploy` to synthesize and deploy the application.

```text
(.venv) Feder08:~/environment/project02 (master) $ cdk deploy
Project02Stack: deploying...
[0%] start: Publishing 4475c6df3f48a28fa9f1799b61a5de7e2472ed14a37b661e38c1d7620bf3ba94:current
[100%] success: Published 4475c6df3f48a28fa9f1799b61a5de7e2472ed14a37b661e38c1d7620bf3ba94:current
Project02Stack: creating CloudFormation changeset...





 ✅  Project02Stack

Outputs:
Project02Stack.MyAPIEndpoint3D9AE6B4 = https://gq36b3n519.execute-api.us-west-2.amazonaws.com/prod/

Stack ARN:
arn:aws:cloudformation:us-west-2:537434832053:stack/Project02Stack/78442d90-e058-11eb-b9ae-06355f45545d
```

The `Outputs:` section tells you the API Gateway URL and the Rust Function can be tested with

```bash
curl -d '{"command":"test"}' https://gq36b3n519.execute-api.us-west-2.amazonaws.com/prod/
```

Remember to substitute the URL that you get as a response from the `cdk deploy` command.
