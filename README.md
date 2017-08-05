# CloudFormation to Terraform

This node CLI tool is used for generating both Terraform configuration files as well as Terraform state so that you can use Terraform to manage CloudFormation templates. To further clarify, it does not generate terraform configuration for the individual resources CloudFormation provisions, it generates an `aws_cloudformation_stack` resource so that you can manage your existing CloudFormation stacks with Terraform instead of or in conjunction with the AWS Console and CLI.

## Getting Started

As this repo isn't published to NPM yet, you will need to clone the repo locally and install it globally:

```
git clone git@github.com:humanmade/cf-to-tf.git cf-to-tf
cd cf-to-tf
npm install -g
```

From there, a `cf-to-tf` command will be available.

### Recommended Dependencies

As this was designed to generate Terraform resources, it'd be a good idea to install [`terraform`](https://www.terraform.io/intro/getting-started/install.html). You can install the binary by itself or use a tool like `brew` to manage it for you.

It's also recommended to install [`json2hcl`](https://github.com/kvz/json2hcl) as this will assist in processing output from `cf-to-tf` later.

## Demo

Let's use the following CloudFormation Stack response as an example:
```
{
    "Stacks": [
        {
            "StackId": "arn:aws:cloudformation:eu-central-1:123456789012:stack/foobarbaz/255491f0-71b8-11e7-a154-500c52a6cefe",
            "Description": "FooBar Stack",
            "Parameters": [
                {
                    "ParameterValue": "bar",
                    "ParameterKey": "foo"
                },
                {
                    "ParameterValue": "baz",
                    "ParameterKey": "bar"
                },
                {
                    "ParameterValue": "qux",
                    "ParameterKey": "baz"

                }
            ],
            "Tags": [
                {
                    "Value": "bar",
                    "Key": "foo"
                },
                {
                    "Value": "qux",
                    "Key": "baz"
                }
            ],
            "Outputs": [
                {
                    "Description": "Foobarbaz",
                    "OutputKey": "FooBarBaz",
                    "OutputValue": "output value"
                }
            ],
            "CreationTime": "2017-07-26T04:08:57.266Z",
            "Capabilities": [
                "CAPABILITY_IAM"
            ],
            "StackName": "foobarbaz",
            "NotificationARNs": [],
            "StackStatus": "CREATE_COMPLETE",
            "DisableRollback": true
        }
    ]
}
```

Running `cf-to-tf --stack foobarbaz config | json2hcl | cf-to-tf clean-hcl | terraform fmt -` will generate the following config:

```
resource "aws_cloudformation_stack" "network" {
  capabilities     = ["CAPABILITY_IAM"]
  disable_rollback = true
  name             = "foobarbaz"

  parameters = {
    foo = "bar"
    bar = "baz"
    baz = "qux"
  }

  tags = {
    foo = "bar"
    baz = "qux"
  }
}
```


## Usage

```
Usage: cf-to-tf [options] [command]


Options:

  -s, --stack <stack>                 The CloudFormation stack to import
  -r, --resource-name <resourceName>  The name to assign the terraform resource
  -h, --help                          output usage information


Commands:

  config      Generates Terraform configuration in JSON
  state       Generates Terraform state file in JSON
  clean-hcl   Cleans generated HCL according to my preferences
```

This tool is designed to be used in conjunction with other tools. It will only output the data to `STDOUT` and is designed to be piped to another program to write the file to a location. For example, to generate a configuration file for a stack named `lambda-resources`, we could do the following:

```
cf-to-tf -s lambda-resources config | tee main.tf.json
```

This command will fetch a CloudFormation stack named `lambda-resources` and generate the required Terraform configuration for it. We then pipe the output to `tee` which will write to a file named `main.tf.json`. Because HCL is JSON compatible, Terraform can read the `main.tf.json` natively.

To generate the associated Terraform state for this CloudFormation stack, you would run the following:

```
cf-to-tf -s lambda-resources state | tee terraform.tfstate
```

This will create a state file from scratch. It assumes you don't already have an existing state file in place. I'm considering updating the tool to write just the resource portion of the state so it can be added to an existing state file, but that wasn't an immediate priority.

Both of these commands will generate compressed JSON output, meaning whitespace has been stripped. To pretty print the output for enhanced readability, you could pipe the output to `jq`, and then to `tee`:

```
cf-to-tf -s lambda-resources config | jq '.' | tee main.tf.json
cf-to-tf -s lambda-resources state | jq '.' | tee terraform.tfstate
```

It's also possible to use a tool called [`json2hcl`](https://github.com/kvz/json2hcl) to generate HCL:

```
cf-to-tf -s lambda-resources config | json2hcl | tee main.tf
```

Unfortunately, while `json2hcl` outputs valid HCL, it's not in the format I like. To solve that problem, the `clean-hcl` command is also available. To output HCL in the format you'll normally see, you can execute this chain:

```
cf-to-tf -s lambda-resources config | json2hcl | cf-to-tf clean-hcl | terraform fmt - | tee main.tf
```

We're doing the same thing we were doing before, but now we're also piping the result to `cf-to-tf clean-hcl` which formats the file a certain way, then piping it to `terraform fmt -` which formats the file further (primarily, this tool aligns `=` and adds newlines where necessary).

The command uses the AWS SDK under the hood to retrieve the CloudFormation stack details, so set your authentication credentials as you would normally (`~/.aws/credentials`, `AWS_PROFILE`, `AWS_REGION`, etc).
