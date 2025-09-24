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

### Ruby Requirements

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

Since we use a template repo for all our Terraform repos, we don't need starter templates. But, we definitely use shared workflows. There are three workflows in [`.github/workflows`](./.github/workflows) that are used in all of our Terraform repos:

- `tf-validate-shared.yml`: we always validate the Terraform code
- `tf-checkov-shared.yml`: we always run a security check on the Terraform code
- `tf-docs-shared.yml`: we always validate that the `README.md` files have the output from the `terraform-docs` command

**The `tf-docs-gen-shared.yml` workflow is deprecated and is replaced by `tf-docs-shared.yml`.**

## Deploying Containers to AWS ECR

There are three workflows associated with publishing Docker containers to AWS ECR.

**Note**: *While these workflows publish containers to ECR they do ***NOT*** force ECS services to restart with the new container.*

These workflows include an input variable that allows the developer to choose either `linux/amd64` or `linux/arm6` as the CPU architecture for the built container.

- `ecr-multi-arch-deploy-dev.yml`: Build/Publish Docker container in dev (for ECS or Lambda)
- `ecr-multi-arch-deploy-stage.yml`: Build/Publish Docker container in stage (for ECS or Lambda)
- `ecr-multi-arch-promote-prod.yml`: Copy Docker container from stage to prod (for ECS or Lambda)

The caller workflows are generated programmatically by the [mitlib-tf-workloads-ecr](https://github.com/MITLibraries/mitlib-tf-workloads-ecr) repository and then are copy/pasted from the Terraform Cloud outputs into the calling application repository.

### ecr-multi-arch-deploy-dev.yml & ecr-multi-arch-deploy-stage.yml

These two workflows are the same. They require

- shared secrets that exists in our MITLibraries GitHub Organization
- a caller workflow that passes expected values to the shared workflow

The container that is pushed to the AWS ECR Repository in Dev1 is tagged with

- the short (8 character) SHA of the most recent commit on the feature branch that generated the PR
- the PR number for the repo
- the word `latest` (or `latest-arm64` or `latest-amd64`)

The container that is pushed to the AWS ECR Repository in Stage-Workloads is tagged with

- the short (8 character) SHA of the merge commit
- the word `latest` (or `latest-arm64` or `latest-amd64`)

The workflow requires a `CPU_ARCH` input. This is used to pick the GitHub-hosted runner architecture for the whole job and then it is used during the Docker build step to build the container for the correct architecture. See [partner-runner-images](https://github.com/actions/partner-runner-images?tab=readme-ov-file) for information related to `ARM64` based GitHub-hosted runners (the default GitHub-hosted runner is based on `AMD64`).

### ecr-multi-arch-promote-prod.yml

The promote to prod workflow just copies the container from stage to prod to ensure that there is no difference at all in what is deployed.

The caller workflows for this shared workflow are configured with `on.workflow_dispatch` and `on.release.types: [published]`. The former allows for manual deploys from the GitHub UI and the latter is for deploying to production when a new release tag is issued. But, we want to ensure that only tagged releases on the `main` branch of the calling repository will trigger a run (we don't want a published tag on a feature branch to unintentionally push something to Production). So, we add a conditional for the `job.deploy` that checks the calling branch:

```yaml
    if: ${{ github.ref == 'refs/heads/main' || github.event.release.target_commitish == 'main' }}
```

The `github.ref` value gets set when the trigger is a release tag. The `github.event.release.target_commitish` value gets set when the trigger is `workflow_dispatch`.

Additionally, we need to ensure that the container in the Stage-Workloads ECR repository is from the merge commit on `main`. (Since it is allowable for a developer to manually run the GitHub Actions Workflow to push a feature branch container to Stage-Workloads, we need to include this verification.) In essence, we are checking that the 8-character SHA tag on the container image in ECR matches the 8-character SHA for the merge commit into `main`.

The container that is pushed to the AWS ECR Repository in Prod is tagged with

- the short (8 character) SHA of the tagged merge commit
- the release tag name
- the word `latest` (or `latest-arm64` or `latest-amd64` or `MANUAL_TRIGGER`)

### Additional Requirements/Dependencies

It also assumes that the appropriate infrastructure is in place, particularly the OIDC configuration and IAM role for Github Actions to connect to AWS. In most cases, this is handled by the [mitlib-tf-workloads-ecr](https://github.com/mitlibraries/mitlib-tf-workloads-ecr) repository. That same repository generates the GHA workflow files for each ECR repository so that they can be copied from Terraform Cloud into the application repository.

## DEPRECATED: Build and Publish Containers

**NOTE: This is deprecated and is being replaced by the preceeding section.**

There are three workflows associated with publishing Docker containers. **Note**: The automated workflows deploy the updated container to the ECR repository, but the workflows **do not** force a container-based service to restart!

- For build/push/update of Docker container in dev (either for ECS or for Lambda): [ecr-shared-deploy-dev.yml](.github/workflows/ecr-shared-deploy-dev.yml)
- For build/push/update of Docker container in stage (either for ECS or for Lambda): [ecr-shared-deploy-stage.yml](.github/workflows/ecr-shared-deploy-stage.yml)
- For promoting container to prod (either for ECS or for Lambda): [ecr-shared-promote-prod.yml](.github/workflows/ecr-shared-promote-prod.yml)

The workflows include checks for the type of container to be built and can handle

- Basic Docker containers (e.g., modifications of containers published on Docker Hub)
- Custom, Python-based containers built by MIT Libraries
- Custom, Ruby-based containers build by MIT Libraries

### DEPRECATED: ecr-shared-deploy-dev.yml & ecr-shared-deploy-stage.yml

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

### DEPRECATED: ecr-shared-promote-prod.yml

The automated deploys to dev & stage actually build the container through GitHub Actions. The promote to prod workflow just copies the container from stage to prod to ensure that there is no difference at all in what is deployed.

The caller workflows for this shared workflow are configured with `on.workflow_dispatch` and `on.release.types: [published]`. The former is for manual deploys from the GitHub UI and the latter is for deploying to production when a new release tag is issued. But, we want to ensure that only tagged releases on the `main` branch of the calling repository will trigger a run (we don't want a published tag on a feature branch to unintentionally push something to Production). So, we add a conditional for the `job.deploy` that checks the calling branch:

```yaml
    if: ${{ github.ref == 'refs/heads/main' || github.event.release.target_commitish == 'main' }}
```

The `github.ref` value gets set when the trigger is a release tag. The `github.event.release.target_commitish` value gets set when the trigger is `workflow_dispatch`.

Additionally, we need to ensure that the container in the Stage-Workloads ECR repository is from the merge commit on `main`. (Since it is allowable for a developer to manually run the GitHub Actions Workflow to push a feature branch container to Stage-Workloads, we need to include this verification.) In essence, we are checking that the 8-character SHA tag on the container image in ECR matches the 8-character SHA for the merge commit into `main`.

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

- the short (8 character) SHA of the tagged merge commit
- the release tag name
- the word "latest"

### DEPRECATED: Additional Requirements/Dependencies

It also assumes that the appropriate infrastructure is in place, particularly the OIDC configuration and IAM role for Github Actions to connect to AWS. In most cases, this is handled by the [mitlib-tf-workloads-ecr](https://github.com/mitlibraries/mitlib-tf-workloads-ecr) repository. That same repository generates the GHA workflow files for each ECR repository so that they can be copied from Terraform Cloud into the application repository.

## Automated Publishing to CDN

There are multiple static HTML repositories (future-of-libraries and open-access-task-force) that will benefit from automated publishing to the S3-based CDN in our AWS Organization. The publishing automation (for both stage & prod) is handled by one shared workflow, [cdn-shared-publish.yml](./.github/workflows/cdn-shared-publish.yml), that covers all three tiers (dev/stage/prod) as well as both the standard CDN and the custom domain CDN.

This workflow assumes that the calling repository is structured in a very particular way!

- For custom domain repos, all the content to be published to the `<folder_name>` folder in the S3 bucket **must** live at the root of the repository.
- For standard domain repos, all the content to be published to the `cdn/<folder_name>` folder in the S3 bucket **must** live in a top level folder named `<folder_name>`. 
  - For a custom domain example see [future-of-libraries-static](https://github.com/mitlibraries/future-of-libraries-static).
  - For a standard CDN example see [web-images-static](https://github.com/mitlibraries/web-images-static).

### CDN Requirements

The following values must be passed in to the shared workflow from the caller workflow:

- `AWS_REGION` (*string*, **required**): the region where the S3 bucket lives
- `DOMAIN` (*string*, **optional**): the default value is `standard` which refers to the standard CDN. If the content in question is associated with the custom domain CDN, then the caller workflow must pass the value `custom` instead of relying on the default.
- `ENVIRONMENT` (*string*, **required**): either `stage` or `prod` (this workflow is not intended for the `dev` environment)
- `GHA_ROLE` (*string*, **required**): the OIDC role (managed by the [mitlib-tf-workloads-libraries-website](https://github.com/MITLibraries/mitlib-tf-workloads-libraries-website) repository)
- `SYNC_PARAMS` (*string*, **optional**): this is a string that is appended to the `aws s3 sync` command. If nothing is passed from the caller workflow, it is ignored. This is intended to be used for adding additional `--exclude` arguments for any other files/folders in the web content repo that shouldn't be published to the S3 bucket for the site.
  - The typical use for the web dev is to exclude additional top level folders (e.g., `--exclude "docs/*"`) or exclude the top level README (`--exclude "README.md"`).
  - It **can** be used to exclude everything except for one top level folder (e.g., `--exclude "*" --include "use_only_this_folder/*"`).
  - for more details on the additional parameters that can be used for `SYNC_PARAMS` see
    - [AWS CLI s3 reference](https://awscli.amazonaws.com/v2/documentation/api/latest/reference/s3/index.html)
    - [AWS CLI s3 sync reference](https://awscli.amazonaws.com/v2/documentation/api/latest/reference/s3/sync.html)
  - The fixed behavior of this workflow is to ignore the `.gitignore` file, the `.git` directory, and the `.github` directory.
- `S3URI` (*string*, **required**): the full S3 URI (including the path) where the files should be uploaded

To make life easy for the web developers, the [mitlib-tf-workloads-libraries-website](https://github.com/MITLibraries/mitlib-tf-workloads-libraries-website) repository generates the correct caller workflow for the custom domain sites and stores it as a Terraform output in TfCloud. This can be copy/pasted into the repository containing the content to be published to the CDN.

## Automated Lambda@Edge Deployments

There are multiple Lambda@Edge functions in our CloudFront distributions. The Lambda update & deployment as well as the CloudFront re-deployment (via Terraform) are centralized here to make it easier to add additional Lambda functions in the future. See [cf-lambda-deploy.yml](./.github/workflows/cf-lambda-shared-deploy.yml) for the actual workflow. See [Lambda@Edge CloudFront Deployment Model](https://mitlibraries.atlassian.net/l/cp/SP3QNj1s) for an overview of the deployment process.

### Lambda@Edge Requirements

The following values (alphabetical order) must be passed in to the shared workflow from the caller workflow:

- `AWS_REGION` (*string*, **required**): The region where the S3 bucket lives
- `ENVIRONMENT` (*string*, **required**): One of `dev`, `stage`, or `prod`
- `GHA_ROLE` (*string*, **required**): The OIDC role (managed by the [mitlib-tf-workloads-libraries-website](https://github.com/MITLibraries/mitlib-tf-workloads-libraries-website) repository)
- `TF_AUTO_APPLY` (*boolean*, **optional**): (default == `false`) A boolean for whether the triggered plan should also be auto-applied. Setting this to `true` means that the TfC plan, if successful, will be automatically applied with no human interaction.
- `TF_WORKSPACE` (*string*, **required**): The name of the Terraform Cloud Workspace to which GHA should connect
- `UPLOAD_ZIP` (*boolean*, **optional**): (default == `false`) A flag determining whether the zip and upload job should execute or not. Setting this to `true` will force the application packaging and upload to S3 to happen before re-applying the Terraform code in TfC.

The following values must be set in the repository that calls this shared workflow:

- `TF_API_TOKEN` **Repository Secret**: This is the lib-gh-tfc user token stored in LastPass.
- `TF_CLOUD_ORGANIZATION` **Repository Variable**: This is the name of our Terraform Cloud Organization (`MITLibraries`).

**Note**: There is a hard dependency on the caller repo having a `Makefile` and having both `create-zip` and `upload-zip` as `make` commands for this workflow to function properly.

To make life easy for the application developers, the [mitlib-tf-workloads-libraries-website](https://github.com/MITLibraries/mitlib-tf-workloads-libraries-website) repository generates the correct caller workflow for the Lambda application repos and stores it as a Terraform output in TfCloud. This can be copy/pasted into the repository containing the app to be deployed to the CloudFront distribution.
