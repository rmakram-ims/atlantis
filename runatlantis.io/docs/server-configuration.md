# Server Configuration
This documentation explains how to configure the Atlantis server and how to deal
with credentials.

[[toc]]

Configuration for `atlantis server` can be specified via command line flags, environment variables or a YAML config file.
Config file values are overridden by environment variables which in turn are overridden by flags.

## YAML
To use a yaml config file, run atlantis with `--config /path/to/config.yaml`.
The keys of your config file should be the same as the flag, ex.
```yaml
---
gh-token: ...
log-level: ...
```

## Environment Variables
All flags can be specified as environment variables. You need to convert the flag's `-`'s to `_`'s, uppercase all the letters and prefix with `ATLANTIS_`.
For example, `--gh-user` can be set via the environment variable `ATLANTIS_GH_USER`.

To see a list of all flags and their descriptions run `atlantis server --help`

::: warning
The flag `--atlantis-url` is set by the environment variable `ATLANTIS_ATLANTIS_URL` **NOT** `ATLANTIS_URL`.
:::

## AWS Credentials
Atlantis simply shells out to `terraform` so you don't need to do anything special with AWS credentials.
As long as `terraform` commands works where you're hosting Atlantis, then Atlantis will work.
See [https://www.terraform.io/docs/providers/aws/#authentication](https://www.terraform.io/docs/providers/aws/#authentication) for more detail.

### Multiple AWS Accounts
Atlantis supports multiple AWS accounts through the use of Terraform's
[AWS Authentication](https://www.terraform.io/docs/providers/aws/#authentication).

If you're using the [Shared Credentials file](https://www.terraform.io/docs/providers/aws/#shared-credentials-file)
you'll need to ensure the server that Atlantis is executing on has the corresponding credentials file.

If you're using [Assume role](https://www.terraform.io/docs/providers/aws/#assume-role)
you'll need to ensure that the credentials file has a `default` profile that is able
to assume all required roles.

[Environment variables](https://www.terraform.io/docs/providers/aws/#environment-variables) authentication
won't work for multiple accounts since Atlantis wouldn't know which environment variables to execute
Terraform with.

### Assume Role Session Names
Atlantis injects 3 Terraform variables that can be used to dynamically name the assume role
session:

```bash
# Set to the VCS username of who is running the plan command, ex. lkysow
variable "atlantis_user" {
  default = "atlantis_user"
}

# Set to the full name of the repo the pull request is in, ex. runatlantis/atlantis
variable "atlantis_repo" {
  default = "atlantis_repo"
}

# Set to the pull request number, ex. 200
variable "atlantis_pull_num" {
  default = "atlantis_pull_num"
}

# Can be used within the assume_role block for session_name.
provider "aws" {
  assume_role {
    role_arn     = "arn:aws:iam::ACCOUNT_ID:role/ROLE_NAME"
    session_name = "${var.atlantis_user}:${var.atlantis_repo}:${var.atlantis_pull_num}"
  }
}
```

Setting `session_name` allows you to trace where API calls made through Atlantis came from in
CloudWatch.

If you're also using the [S3 Backend](https://www.terraform.io/docs/backends/types/s3.html)
make sure to add the `role_arn` option:

```hcl
terraform {
  backend "s3" {
    bucket   = "mybucket"
    key      = "path/to/my/key"
    region   = "us-east-1"
    role_arn = "arn:aws:iam::ACCOUNT_ID:role/ROLE_NAME"
    # can't use var.atlantis_user as the session name because
    # interpolations are not allowed in backend configuration
    # session_name = "${var.atlantis_user}" WON'T WORK
  }
}
```

::: warning
Terraform doesn't support interpolations in backend config so you will not be
able to use `session_name = "${var.atlantis_user}"`. However, the backend assumed
role is only used for state-related API actions. Any other API actions will be performed using
the assumed role specified in the `aws` provider and will have the session named as the GitHub user.
:::

