# GitHub Actions Workflows

This repository stores both Starter Workflows and re-usable GitHub Actions Workflows for MIT Libraries.

If your actions are only runable in a specific repository, this is not the place to store them.

If your actions are currently copy/pasted across several similar repositories, consider moving them here and updating your other repos to run from the copy here.

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
