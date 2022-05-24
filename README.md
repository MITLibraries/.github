# GitHub Actions Workflows

This repository stores both Starter Workflows and re-usable GitHub Actions Workflows for MIT Libraries.

If your actions are only runable in a specific repository, this is not the place to store them.

If your actions are currently copy/pasted across several similar repositories, consider moving them here and updating your other repos to run from the copy here.

## Python CI

### Requirements

Ensure the repo using these re-usable workflow(s) has all of the following:

- A .python-version file specifying the same Python version as the Pipfile
- A Pipfile with `pytest` and `coverage` in the dev-packages section
- If using the linting workflow, a Makefile with a `make lint` command. See the [oai-pmh-harvester repo Makefile](https://github.com/MITLibraries/oai-pmh-harvester/blob/213d610c2095145b071e7ba730a282c578111579/Makefile) for an example of standard linter usage.

In the repo where you want to use these workflows, add a `ci.yml` file in`.github/workflows` containing the following:

```yaml
name: CI
on: push
jobs:
  test:
    uses: mitlibraries/.github/.github/workflows/python-shared-test.yml@main
  lint:
    uses: mitlibraries/.github/.github/workflows/python-shared-lint.yml@main
```

That's it! Adding the shared workflow should now run the standard tests, code coverage checks, and linting.

## Ruby CI

You can (optionally) add this using a Starter Workflow by going to Actions in the repository you'd like it added to, and selecting the "MIT Libraries Ruby CI Workflow" which will allow you to open a PR to add this shared workflow.

Requirements

- Add `simplecov` and `simplecov-lcov` to the test section of Gemfile
- `bundle install`
- Add the following config to `test/test_helper.rb`

Insert immediately after the ENV check on line 1.

```ruby
require 'simplecov'
 require 'simplecov-lcov'
 SimpleCov::Formatter::LcovFormatter.config.report_with_single_file = true
 SimpleCov::Formatter::LcovFormatter.config.lcov_file_name = 'coverage.lcov'
 SimpleCov.formatters = [
   SimpleCov::Formatter::HTMLFormatter,
   SimpleCov.formatter = SimpleCov::Formatter::LcovFormatter
 ]
 SimpleCov.start('rails')
```

- That's it! Adding the shared workflow should now run the standard tests and code coverage checks.

If you don't want to use the Starter Workflow to add this Action, it is still a good place to see how to call this Shared Workflow.

## Terraform

Since we use a template repo for all our Terraform repos, we don't need starter templates. But, we can definitely use shared workflows! There are three workflows in [`.github/workflows`](./.github/workflows) that are used in all of our Terraform repos:

- `terraform validate`: we always validate the Terraform code
- `checkov`: we always run a security check on the Terraform code
- `terraform-docs`: we automatically update the `README.md` with the output from the `terraform-docs` command

All these shared workflows have `tf-` as a prefix.

## Publish Lambda Container

**WIP**: This is a work-in-progress, but it seems that the workflow automation for deploying updated containers in Lambda functions will be similar across repos. There are two workflows:

- For build/push/update of container & Lambda in dev and stage: [lambda-shared-deploy.yml](.github/workflows/lambda-shared-deploy.yml)
- For build/push/update of container & Lambda in prod: [lambda-shared-promote-prod.yml](.github/workflows/lambda-shared-promote-prod.yml)

### lambda-shared-deploy.yml

This requires

1. certain commands in the Makefile of the Lambda/Container repo
1. a caller workflow that passes certain values to the shared workflow to set the environment in the runner
1. secrets set in the repo (or set as shared secrets for all of MITLibraries GitHub Org)

The `Makefile` header must contain

```makefile
SHELL=/bin/bash
DATETIME:=$(shell date -u +%Y%m%dT%H%M%SZ)
FUNC=$(FUNC_NAME)
ECR_REPO=$(REPO_NAME)
ECR=$(shell aws sts get-caller-identity --query Account --output text).dkr.ecr.us-east-1.amazonaws.com
ECR_PROD=$(AWS_ACCOUNT_ID_PROD).dkr.ecr.us-east-1.amazonaws.com
```

and it must contain the following commands

```makefile
gha-dist: ## Build docker container (only used by GitHub Actions)
  docker build --platform linux/amd64 \
    -t $(ECR)/$(ECR_REPO):latest \
    -t $(ECR)/$(ECR_REPO):`git describe --always` . 

gha-publish: ## Push the dev image to ECR in Stage-Workloads (only used by GitHub Actions)
  docker login -u AWS -p $$(aws ecr get-login-password --region us-east-1) $(ECR)
  docker push $(ECR)/$(ECR_REPO):latest
  docker push $(ECR)/$(ECR_REPO):`git describe --always`

gha-update-lambda: ## Updates the lambda with whatever is the most recent image in the ecr (only used by GitHub Actions)
  aws lambda update-function-code --function-name $(FUNC) --image-uri $(ECR)/$(ECR_REPO):latest
```

A sample caller workflow should look like

```yaml
jobs:
  deploy:
    name: Deploy Container to Lambda in Dev1
    uses: mitlibraries/.github/.github/workflows/lambda-shared-deploy.yml@main
    with:
      GHA_ROLE: "lambda-image-gha-dev"
      REGION: "us-east-1"
      FUNC_NAME: "dev-lambda-image"
      REPO_NAME: "lambda-image"
    secrets:
      AWS_ACCT: ${{ secrets.AWS_DEV1_ACCT }}
```

### lambda-shared-promote-prod.yml

Similar to the previous workflow, there are multiple requirements for the `Makefile` and the caller workflow.

The `Makefile` needs the following commands (the header is the same as above). Note that the first command here is the same as the `gha-update-lambda` command as above.

```makefile
gha-update-lambda: ## Updates the lambda with whatever is the most recent image in the ecr (only used by GitHub Actions)
  aws lambda update-function-code \
    --function-name $(FUNC) \
    --image-uri $(ECR)/$(ECR_REPO):latest

gha-download-stage: ## Get the stage image from ECR in Stage (only used by GitHub Actions)
  docker login -u AWS -p $$(aws ecr get-login-password --region us-east-1) $(ECR)
  docker pull $(ECR)/$(ECR_REPO):latest
  docker tag $(ECR)/$(ECR_REPO):latest $(ECR_PROD)/$(ECR_REPO):latest
  docker tag $(ECR)/$(ECR_REPO):latest $(ECR_PROD)/$(ECR_REPO):`git describe --always`

gha-deploy-prod: ## Copy the downloaded image from runner to Prod ECR (only used by GitHub Actions)
  docker login -u AWS -p $$(aws ecr get-login-password --region us-east-1) $(ECR_PROD)
  docker push $(ECR_PROD)/$(ECR_REPO):latest
  docker push $(ECR_PROD)/$(ECR_REPO):`git describe --always`
```

A sample caller workflow would look like

```yaml
jobs:
  deploy:
    name: Promote
    uses: mitlibraries/.github/.github/workflows/lambda-shared-promote-prod.yml@main
    with:
      GHA_ROLE_STAGE: "lambda-image-gha-stage"
      GHA_ROLE_PROD: "lambda-image-gha-prod"
      REGION: "us-east-1"
      FUNC: "prod-lambda-image"
      REPO: "lambda-image"
    secrets:
      AWS_ACCT_STAGE: ${{ secrets.AWS_STAGE_ACCT }}
      AWS_ACCT_PROD: ${{ secrets.AWS_PROD_ACCT }}
```

### Additional Requirements/Dependencies

It also depends on the appropriate infrastructure in place, particularly the OIDC configuration and IAM role for Github Actions to connect to AWS.

## Publish Fargate Container

**WIP**: This hasn't started yet.
