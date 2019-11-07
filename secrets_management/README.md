## Secrets Management

We manage our application secrets securely by storing them in [AWS Parameter Store][1]
and reading and writing them using [chamber][2].

### Initial Setup

Ensure you have installed and configured the AWS cli ([installation instructions here][3])
and `aws-vault` ([installation instructions here][4]).

Then install chamber:

* download the latest binary for your platform from https://github.com/segmentio/chamber/releases
* For Mac/Linux `chmod a+rwx` the downloaded binary and symlink to `/usr/local/bin/chamber`
* On Windows install using the `.exe`

### Application setup and usage

#### aws-vault

If you have individual AWS account keys, just add a profile for that account in aws-vault, e.g.:

    $ aws-vault add account-name

`account-name` can be any name you want - it is what you provide to aws-vault to use this role.

If you have [organization](https://aws.amazon.com/organizations/) account keys, it is not necessary to get individual credentials for each account, instead aws-vault can be used to switch roles to the account and provide temporary credentials to access the state.

First, ensure you have added your Organisation AWS account access key and secret key as an aws-vault profile:

    $ aws-vault add org-account

`org-account` can be any name you want, but it must be used as the `source_profile` below.

Then, add a section to your [aws config file](http://docs.aws.amazon.com/cli/latest/userguide/cli-config-files.html) for each environment account:

    [profile account-name]
    role_arn = arn:aws:iam::<account id>:role/OrganizationAccountAccessRole
    source_profile = org-account

`eventcipher-staging` can be any name you want - it is what you provide to aws-vault to use this role. The default role for accessing org accounts on AWS is `OrganizationAccountAccessRole`, however
it is possible to define an alternative for the account, so you may need to confirm the role name.

Once this is in place, you can then use aws-vault to obtain temporary credentials.

    $ aws-vault exec account-name -- <command>

`aws-vault exec <profile>` will insert the AWS credentials for `<profile>` into the current environment. You can view the environment variables that have been created like this (in bash):

    $ aws-vault exec <profile> -- env | grep AWS

This can be very useful to debug. You can delete the sessions for a profile with

    $ aws-vault remove -s <profile>

### chamber

Chamber is a wrapper around AWS SSM parameter store, which enables us to read and write encrypted secrets. [usage docs](https://github.com/segmentio/chamber#usage). We chain it with aws-vault to insert the decrypted secrets as temporary environment variables:

    $ aws-vault exec <profile> -- chamber exec <service...> -- <command>

Multiple chamber services can be chained to provide inheritance. It may be convenient to alias commonly used chains.

Chamber doesn't provide wrappers for every SSM parameter store action, in which case you can drop down to the aws cli.

To see what "services" are available to you through chamber:

    $ aws-vault exec <profile> -- aws ssm get-parameters-by-path --path /

The first portion of the `Name` key of each parameter object is what chamber requires as a service name. The full value of that `Name` key for a parameter can be used to delete it:

    $ aws-vault exec <profile> -- aws ssm delete-parameter --name "<name>"

This will delete the history for that secret as well.


[1][https://aws.amazon.com/systems-manager/features/#Parameter_Store]
[2][https://github.com/segmentio/chamber]
[3][https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-install.html]
[4][https://github.com/99designs/aws-vault#installing]
[5][https://github.com/segmentio/chamber/wiki/Installation#macos-binaries]
