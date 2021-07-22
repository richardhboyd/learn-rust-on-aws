# Project 03.01 - CDK

We'll start by making sure Rust is installed and using the nightly toolchain.

```bash
curl https://sh.rustup.rs -sSf | sh -s -- -y --default-toolchain nightly
source $HOME/.cargo/env
```

I'll explain later in the guide why we're using the nightly toolchain.

Next we'll create a directory to hold our CDK application.

```bash
mkdir project01
cd project01/
```

Now we'll create a scaffolded CDK application.

```bash
Feder08:~/environment/project01 $ cdk init 
```

This will output a bunch of text to tell you about the available options for `cdk init`

```text
Available templates:
* app: Template for a CDK Application
   └─ cdk init app --language=[csharp|fsharp|go|java|javascript|python|typescript]
* lib: Template for a CDK Construct Library
   └─ cdk init lib --language=typescript
* sample-app: Example CDK Application with some constructs
   └─ cdk init sample-app --language=[csharp|fsharp|java|javascript|python|typescript]
****************************************************
*** Newer version of CDK is available [1.108.0]  ***
*** Upgrade recommended (npm install -g aws-cdk) ***
**********************************c
```

It also shows a message about a newer version being available (`1.108.0` here).
CDK generally releases a new version about once every 10 days, let's check what version we have to see how important it is that we upgrade.

```text
Feder08:~/environment/project01 $ cdk --version
1.107.0 (build 52c4434)
```

Just one version behind.
This isn't too bad, I'm not going to bother to update.
I would say you generally want to be within about 10 minor version of the most recent.

Let's go ahead and create a new `app`.
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
I would describe our current state of the application as similar to the moment after you run `cargo new`.

This might be a good time to explain what all of these files are for and why the CDK needs/uses them.

TODO: explain all these files and packages

First, we'll create our Lambda function code.

## Lambda Function Code

run `cargo new my-example-function` to create a folder named `my-example-function` and replace the contents of the following files.

<details>
  <summary>Cargo.toml</summary>
  
```toml
[package]
name = "my-example-function"
version = "0.1.0"
edition = "2018"

[dependencies]
tokio = { version = "1", features = ["full"] }
serde = { version = "1.0.82", features = ["derive"] }
serde_json = { version = "1.0.33", features = ["raw_value"] }

lambda_runtime = "0.3"

[[bin]]
name = "bootstrap"
path = "src/main.rs"
```

</details>

<details>
  <summary>src/main.rs</summary>
  
```rust
use lambda_runtime::{handler_fn, Context};
use serde::{Deserialize, Serialize};
// use serde_json::Result;

pub type Error = Box<dyn std::error::Error + Send + Sync + 'static>;

#[derive(Deserialize)]
struct APIRequest {
    body: String,
}

#[derive(Serialize, Deserialize)]
struct Request {
    command: String,
}

#[derive(Serialize)]
struct Response {
    statusCode: String,
    body: String,
}

#[tokio::main]
async fn main() -> Result<(), Error> {
    let func = handler_fn(my_handler);
    lambda_runtime::run(func).await?;
    Ok(())
}

pub(crate) async fn my_handler(event: APIRequest, ctx: Context) -> Result<Response, Error> {
    let r: Request = serde_json::from_str(&event.body)?;
    let command = r.command;

    let resp = Response {
        statusCode: "200".to_string(),
        body: format!("Command {} executed.", command),
    };

    Ok(resp)
}
```

</details>

Now we'll build this binary with the following command:

`cargo +nightly build --release --out-dir=./my-example-function/out -Z unstable-options --manifest-path ./my-example-function/Cargo.toml `

This will place all of the build artifacts in the usual `target/release/` directory, but it will also copy just the binary to a directory named `out/`.
This will be important for the later CDK step.

## CDK Specifics

First, we will update our CDK application to take a dependency on the CDK module for AWS Lambda (in CDK V2, this won't be necessary, but we'll discuss that later)

add `aws-cdk.aws-lambda==X.YY.ZZ` to `setup.py` within `install_requires` section.

```python
    install_requires=[
        "aws-cdk.core==1.107.0",
        "aws-cdk.aws-lambda==1.107.0",
        "aws-cdk.aws-apigateway==1.107.0"
    ],
```

Install the Python dependencies

```text
pip install -r ./requirements.txt
```

Update the `project01_stack.py` file to create a Lambda Function.

```python
from aws_cdk import core as cdk
import aws_cdk.aws_lambda as lambda_
import aws_cdk.aws_apigateway as apigw

class Project01Stack(cdk.Stack):

    def __init__(self, scope: cdk.Construct, construct_id: str, **kwargs) -> None:
        super().__init__(scope, construct_id, **kwargs)

        function_reference = lambdaFn = lambda_.Function(
            self, "Singleton",
            code=lambda_.Code.from_asset('./my-example-function/out'),
            handler="index.main",
            runtime=lambda_.Runtime.PROVIDED_AL2,
        )
        apigw.LambdaRestApi(self, "MyAPI", handler=function_reference)
```

Note that the `code` parameter is `lambda_.Code.from_asset('./my-example-function/out')`, which means that we're telling CDK to grab the contents of a directory, zip it up, and use that as the function's 'code artifact'.
If we hadn't used the `out-dir` feature, we would have had to either (a) point to the `target/release/` directory which would bring all of the build artifacts with it or (b) manually move the `bootstrap` executable to a new directory.
Using `out-dir` helps us avoid having to do that my telling Cargo to move the executable for us.

now we're ready to run `cdk synth` (to see the synthesized CloudFormation stack) or `cdk deploy` to synthesize and deploy the application.

```bash
cdk deploy
```

You may be prompted to bootstrap the account if the deploy command fails with the following message
```text
Do you wish to deploy these changes (y/n)? y
Project01Stack: deploying...

 ❌  Project01Stack failed: Error: This stack uses assets, so the toolkit stack must be deployed to the environment (Run "cdk bootstrap aws://unknown-account/unknown-region")
This stack uses assets, so the toolkit stack must be deployed to the environment (Run "cdk bootstrap aws://unknown-account/unknown-region")
```

This can be fixed by running `cdk bootstrap`

```text
(.venv) Feder08:~/environment/project01 (master) $ cdk bootstrap
 ⏳  Bootstrapping environment aws://537434832053/us-west-2...
CDKToolkit: creating CloudFormation changeset...

 ✅  Environment aws://537434832053/us-west-2 bootstrapped.
```

Now you can run `cdk deploy` again.

```text
(.venv) Feder08:~/environment/project01 (master) $ cdk deploy
Project01Stack: deploying...
[0%] start: Publishing 4475c6df3f48a28fa9f1799b61a5de7e2472ed14a37b661e38c1d7620bf3ba94:current
[100%] success: Published 4475c6df3f48a28fa9f1799b61a5de7e2472ed14a37b661e38c1d7620bf3ba94:current
Project01Stack: creating CloudFormation changeset...





 ✅  Project01Stack

Outputs:
Project01Stack.MyAPIEndpoint3D9AE6B4 = https://gq36b3n519.execute-api.us-west-2.amazonaws.com/prod/

Stack ARN:
arn:aws:cloudformation:us-west-2:537434832053:stack/Project01Stack/78442d90-e058-11eb-b9ae-06355f45545d
```

The `Outputs:` section tells you the API Gateway URL and the Rust Function can be tested with

```bash
curl -d '{"command":"test"}' https://gq36b3n519.execute-api.us-west-2.amazonaws.com/prod/
```

Remember to substitute the URL that you get as a response from the `cdk deploy` command.
