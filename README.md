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

### Requirements

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

## Build and Publish Containers

There are three workflows associated with publishing Docker containers. **Note**: The automated workflows deploy the updated container to the ECR repository, but the workflows **do not** force a container-based service to restart!

- For build/push/update of Docker container in dev: [ecr-shared-deploy-dev.yml](.github/workflows/fargate-shared-deploy-dev.yml)
- For build/push/update of Docker container in stage: [ecr-shared-deploy-stage.yml](.github/workflows/fargate-shared-deploy-stage.yml)
- For promoting container to prod: [ecr-shared-promote-prod.yml](.github/workflows/fargate-shared-promote-prod.yml)

The workflows include checks for the type of container to be built and can handle

- Basic Docker containers (e.g., modifications of containers published on Docker Hub)
- Custom, Python-based containers built by MIT Libraries
- Custom, Ruby-based containers build by MIT Libraries

### ecr-shared-deploy-dev.yml & ecr-shared-deploy-stage.yml

These two workflows are almost exactly the same. They require

- shared secrets that exists in our MITLibraries GitHub Organization.
- a caller workflow that passes certain values to the shared workflow

A sample caller workflow at a minimum, needs

```yaml
jobs:
  deploy:
    name: <env> Container Deploy
    uses: mitlibraries/.github/.github/workflows/ecr-shared-deploy-<env>.yml@main
    secrets: inherit
    with:
      AWS_REGION: "<aws_region>"
      GHA_ROLE: "<name_of_iam_role_with_ecr_publish_permissions>"
      ECR: "<name_of_ecr_repository>"
```

There are two optional `with:` arguments:

```yaml
    with:
      FUNCTION: "${function}"
      PREBUILD: make deps
```

If the application repo is building a container for a Lambda function, the developer must include the `FUNCTION: "${function}"` line. If the application (Python or Ruby) has additional pre-build commands that must be run before the `docker-build` command is run, they need to be added to the `PREBUILD"` argument. It is best if these are handled by the `Makefile`/`Rakefile`, so that the `PREBUILD:` line is just a list of `make`/`rake` commands.

The container that is pushed to the AWS ECR Repository in Dev1 is tagged with

- the short (8 character) SHA of the most recent commit on the feature branch that generated the PR
- the PR number for the repo
- the word "latest"

The container that is pushed to the AWS ECR Repository in Stage-Workloads is tagged with

- the short (8 character) SHA of the merge commit
- the word "latest"

### ecr-shared-promote-prod.yml

The automated deploys to dev & stage actually build the container through GitHub Actions. The promote to prod workflow just copies the container from stage to prod to ensure that there is no difference at all in what is deployed.

A sample caller workflow would look like

```yaml
jobs:
  deploy:
    name: Prod Container Promote 
    uses: mitlibraries/.github/.github/workflows/ecr-shared-promote-prod.yml@main
    secrets: inherit
    with:
      AWS_REGION: "<aws_region>"
      GHA_ROLE_STAGE: "<name_of_iam_role_with_ecr_publish_permissions_in_stage>"
      GHA_ROLE_PROD: "<name_of_iam_role_with_ecr_publish_permissions_in_prod>"
      ECR_STAGE: "<name_of_ecr_repository_in_stage>"
      ECR_PROD: "<name_of_ecr_repository_in_prod>"
```

There is one optional `with:` argument: `FUNCTION: "${function}"` in the situation where the container is destined for a Lambda Function.

The container that is pushed to the AWS ECR Repository in Prod is tagged with

- the short (8 character) SHA of the most recent commit on the feature branch that generated the PR
- the release tag
- the word "latest"

### Additional Requirements/Dependencies

It also depends on the appropriate infrastructure in place, particularly the OIDC configuration and IAM role for Github Actions to connect to AWS. In most cases, this is handled by the [mitlib-tf-workloads-ecr](https://github.com/mitlibraries/mitlib-tf-workloads-ecr) repository (which also generates the GHA workflow files for each ECR repository).

## Automated Publishing to CDN

There are multiple static HTML repositories (future-of-libraries and open-access-task-force) that will benefit from automated publishing to the S3-based CDN in our AWS Organization. The publishing automation (for both stage & prod) is handled by one shared workflow, [cdn-shared-publish.yml](./.github/workflows/cdn-shared-publish.yml), that covers all three tiers (dev/stage/prod) as well as both the standard CDN and the custom domain CDN.

This workflow assumes that the calling repository is structured in a very particular way!

- For custom domain repos, all the content to be published to the `<folder_name>` folder in the S3 bucket **must** live at the root of the repository.
- For standard domain repos, all the content to be published to the `cdn/<folder_name>` folder in the S3 bucket **must** live in a top level folder named `<folder_name>`. 
  - For a custom domain example see [future-of-libraries-static](https://github.com/mitlibraries/future-of-libraries-static).
  - For a standard CDN example see [web-images-static](https://github.com/mitlibraries/web-images-static).

### Requirements

The following values must be passed in to the shared workflow from the caller workflow:

- `AWS_REGION`: the region where the S3 bucket lives
- `GHA_ROLE`: the OIDC role (managed by the [mitlib-tf-workloads-libraries-website](https://github.com/MITLibraries/mitlib-tf-workloads-libraries-website) repository)
- `ENVIRONMENT`: either `stage` or `prod` (this workflow is not intended for the `dev` environment)
- `S3URI`: the full S3 URI (including the path) where the files should be uploaded

There are two optional `with:` arguments:

- `DOMAIN`: the default value is `standard` which refers to the standard CDN. If the content in question is associated with the custom domain CDN, then the caller workflow must pass the value `custom` instead of relying on the default.
- `SYNC_PARAMS`: this is a string that is appended to the `aws s3 sync` command. If nothing is passed from the caller workflow, it is ignored. This is intended to be used for adding additional `--exclude` arguments for any other files/folders in the web content repo that shouldn't be published to the S3 bucket for the site.
  - The typical use for the web dev is to exclude additional top level folders (e.g., `--exclude "docs/*"`) or exclude the top level README (`--exclude "README.md"`).
  - It **can** be used to exclude everything except for one top level folder (e.g., `--exclude "*" --include "use_only_this_folder/*"`).
  - for more details on the additional parameters that can be used for `SYNC_PARAMS` see
    - [AWS CLI s3 reference](https://awscli.amazonaws.com/v2/documentation/api/latest/reference/s3/index.html)
    - [AWS CLI s3 sync reference](https://awscli.amazonaws.com/v2/documentation/api/latest/reference/s3/sync.html)
  - The fixed behavior of this workflow is to ignore the `.gitignore` file, the `.git` directory, and the `.github` directory.

To make life easy for the web devs, the [mitlib-tf-workloads-libraries-website](https://github.com/MITLibraries/mitlib-tf-workloads-libraries-website) repository generates the correct caller workflow for the custom domain sites and stores it as a Terraform output in TfCloud. This can be copy/pasted into the repository containing the content to be published to the CDN.
