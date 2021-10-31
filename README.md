# CloudFormation template for CI/CD pipeline using Terraform

Many companies are looking for automation patterns using Terraform.
Some of them can use Terraform Cloud, but many of them, due to security reasons are not allowed to do so as they require to stay within AWS ecosystem.

The template I prepared for you solves that problem by deploying ready-to-go infrastructure for running Terraform CI/CD pipeline in AWS.

The template provisions:
* CodeCommit repository for storing Terraform
* CodePipeline pipeline for running CI/CD with CodeBuild project as the execution component
* S3 Bucket for Terraform state storage
* DynamoDB table for locking of Terraform state
* Additional resources required for running the solution such as roles for the services or EventBridge rules for invoking the pipeline on commits to CodeCommit repository.

---
**WARNING**

This template is running Terraform in auto-approve mode so all changes will be applied to the infrastructure immediately.

---

## Setup

1. Deploy CloudFormation template using [AWS Console](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/cfn-console-create-stack.html), [AWS CLI](https://docs.aws.amazon.com/cli/latest/reference/cloudformation/deploy/index.html) or using a nice tool like [rain](https://github.com/aws-cloudformation/rain).
2. Optionally provide additional parameters (see below)
3. The stack will output sample templates for Terraform configuration:

```
~/projects/cf-tf-pipeline$ rain deploy pipeline.yaml test -y
Deploying template 'pipeline.yaml' as stack 'test' in eu-west-1.
Stack test: UPDATE_COMPLETE
  Outputs:
    TerraformProviderTemplate: # put this to provider.tf in the repository
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "3.63.0"
    }
  }
}

provider "aws" {
  region              = "eu-west-1"
  allowed_account_ids = ["123456789012"]
  assume_role {
    role_arn = "arn:aws:iam::123456789012:role/terraform-test"
  }
}

    CloneUrlSsh: git clone ssh://git-codecommit.eu-west-1.amazonaws.com/v1/repos/terraform-test # Shell command for connecting to the CodeCommit repo, make sure to configure your ssh for AWS CodeCommit first
    TerraformBackendTemplate: # put this to terraform_backend.tf in the repostiroy
terraform {
  backend "s3" {
    bucket         = "terraform-state-test-0a2b0db68ccf"
    key            = "main"
    region         = "eu-west-1"
    role_arn       = "arn:aws:iam::123456789012:role/terraform-test"
    dynamodb_table = "terraform-lock-test"
  }
}
Successfully updated test
```
4. Clone CodeBuild repository using the `CloneUrlSsh` URL provided by template output. Keep in mind you need to [confiugre credentials for SSH clone](https://docs.aws.amazon.com/codecommit/latest/userguide/setting-up-ssh-unixes.html) before attemping to clone.

5. Create `provider.tf` file inside clonned directory and put the contents provided by `TerraformProviderTemplate` output value from the stack.

6. Create `terraform_backend.tf` file inside clonned directory  and put the contents provided by `TerraformBackendTemplate` output value from the stack.

7. Commit the changes and push the changes do CodeCommit repository.

8. The pipeline will be executed and your changes applied.

## Parameters

There are some optional parameters you can provide to the stack template:

`TerraformVersion` - specifies the version that will be used during execution, the format is just a version number like `1.0.10`.

`TrustedAwsPrincipals` - if you want to use Terraform from your workstation, you can put your prinicipal ARNs here as a comma separated list of ARNs. This will enable assume role functionality for local Terraform runs.