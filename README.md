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

## Publish Fargate Container

There are three workflows associated with publishing Docker containers. **Note**: The automated workflows deploy the updated container to the ECR repository, but the workflows **do not** force a container-based service to restart!

- For build/push/update of Docker container in dev: [fargate-shared-deploy-dev.yml](.github/workflows/fargate-shared-deploy-dev.yml)
- For build/push/update of Docker container in stage: [fargate-shared-deploy-stage.yml](.github/workflows/fargate-shared-deploy-stage.yml)
- For promoting container to prod: [fargate-shared-promote-prod.yml](.github/workflows/fargate-shared-promote-prod.yml)

### fargate-shared-deploy-dev.yml & fargate-shared-deploy-stage.yml

These two workflows are almost exactly the same. They require

- shared secrets that exists in our MITLibraries GitHub Organization.
- a caller workflow that passes certain values to the shared workflow

A sample caller workflow should look like

```yaml
jobs:
  deploy:
    name: <env> Deploy Fargate Container
    uses: mitlibraries/.github/.github/workflows/fargate-shared-deploy-<env>.yml@main
    secrets: inherit
    with:
      AWS_REGION: "<aws_region>"
      GHA_ROLE: "<name_of_iam_role_with_ecr_publish_permissions>"
      ECR: "<name_of_ecr_repository>"
```

The container that is pushed to the AWS ECR Repository in Dev1 is tagged with

- the short (8 character) SHA of the most recent commit on the feature branch that generated the PR
- the PR number for the repo
- the word "latest"

The container that is pushed to the AWS ECR Repository in Stage-Workloads is tagged with

- the short (8 character) SHA of the merge commit
- the word "latest"

### fargate-shared-promote-prod.yml

The automated deploys to dev & stage actually build the container through GitHub Actions. The promote to prod workflow just copies the container from stage to prod to ensure that there is no difference at all in what is deployed.

A sample caller workflow would look like

```yaml
jobs:
  deploy:
    name: Prod Promote Fargate Container
    uses: mitlibraries/.github/.github/workflows/fargate-shared-promote-prod.yml@main
    secrets: inherit
    with:
      AWS_REGION: "<aws_region>"
      GHA_ROLE_STAGE: "<name_of_iam_role_with_ecr_publish_permissions_in_stage>"
      GHA_ROLE_PROD: "<name_of_iam_role_with_ecr_publish_permissions_in_prod>"
      ECR_STAGE: "<name_of_ecr_repository_in_stage>"
      ECR_PROD: "<name_of_ecr_repository_in_prod>"
```

The container that is pushed to the AWS ECR Repository in Prod is tagged with

- the short (8 character) SHA of the most recent commit on the feature branch that generated the PR
- the release tag
- the word "latest"

### Additional Requirements/Dependencies

It also depends on the appropriate infrastructure in place, particularly the OIDC configuration and IAM role for Github Actions to connect to AWS.

**Note**: The caller workflows are generated by the Terraform code that creates the ECR and associated IAM Roles and permissions. It is up to the engineer to copy that text from the Terraform Cloud outputs to the application repository.

## Publish Lambda Container

This is almost exactly the same as the Fargate workflows. The only difference is the inclusion of the name of the Lambda function itself. See the Fargate section above for full details.

### lambda-shared-deploy-dev.yml and lambda-shared-deploy-stage.yml

A sample caller workflow should look like

```yaml
jobs:
  deploy:
    name: <env> Deploy lambda Container
    uses: mitlibraries/.github/.github/workflows/lambda-shared-deploy-<env>.yml@main
    secrets: inherit
    with:
      AWS_REGION: "<aws_region>"
      GHA_ROLE: "<name_of_iam_role_with_ecr_publish_permissions>"
      ECR: "<name_of_ecr_repository>"
      FUNCTION: "<name_of_lambda_function>"
```

### lambda-shared-promote-prod.yml

This is almost exactly the same as for Fargate containers, just with the addition of the Lambda function name.

A sample caller workflow would look like

```yaml
jobs:
  deploy:
    name: Prod Promote Lambda Container
    uses: mitlibraries/.github/.github/workflows/lambda-shared-promote-prod.yml@main
    secrets: inherit
    with:
      AWS_REGION: "<aws_region>"
      GHA_ROLE_STAGE: "<name_of_iam_role_with_ecr_publish_permissions_in_stage>"
      GHA_ROLE_PROD: "<name_of_iam_role_with_ecr_publish_permissions_in_prod>"
      ECR_STAGE: "<name_of_ecr_repository_in_stage>"
      ECR_PROD: "<name_of_ecr_repository_in_prod>"
      FUNCTION: "<name_of_lambda_function_in_prod>"
```

### Additional Requirements/Dependencies

It also depends on the appropriate infrastructure in place, particularly the OIDC configuration and IAM role for Github Actions to connect to AWS.

**Note**: The caller workflows are generated by the Terraform code that creates the ECR and associated IAM Roles and permissions. It is up to the engineer to copy that text from the Terraform Cloud outputs to the application repository.
