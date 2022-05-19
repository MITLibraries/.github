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

* `terraform validate`: we always validate the Terraform code
* `checkov`: we always run a security check on the Terraform code
* `terraform-docs`: we automatically update the `README.md` with the output from the `terraform-docs` command

All these shared workflows have `tf-` as a prefix.

## Publish Lambda Container - DEV

**WIP**
This is a work-in-progress, but it seems that the workflow automation for deploying updated containers in Lambda functions will be similar across repos. There is one workflow [lambda-shared-deploy-dev.yml](./github/workflows/lambda-shared-deploy-dev.yml) that should work for automated deploys in the Dev1 environment. It requires that the caller workflow pass three values through to the called workflow. A sample caller workflow would look like

```yaml
name: Deploy Lambda to Dev
on:
  push:
    branch:
      - 'main'

jobs:
  deploy:
    name: Deploy Container to Lambda
    uses: mitlibraries/.github/.github/workflows/lambda-shared-deploy-dev.yml@main
    with:
      GHA_ROLE: "<name_of_role_provided_by_infraeng>"
      REGION: "<name_of_AWS_region>"
      ACCOUNT: "<AWS_account_number_for_dev>"

```

### Requirements/Dependencies

This depends on the app repo having a Makefile with the appropriate commands:

- `make dist-dev`
- `make publish-dev`
- `make update-lambda-dev`

It also depends on the appropriate infrastructure in place, particularly the OIDC configuration and IAM role for Github Actions to connect to AWS.
