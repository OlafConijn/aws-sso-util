# aws-sso-util
## Making life with AWS SSO a little easier

[AWS SSO](https://aws.amazon.com/single-sign-on/) has some rough edges, and `aws-sso-util` is here to smooth them out, hopefully temporarily until AWS makes it better.

`aws-sso-util` contains utilities for the following:
* Configuring `.aws/config`
* Logging in/out
* AWS SDK support
* Looking up identifiers
* CloudFormation

The underlying Python library for AWS SSO authentication is `aws-sso-lib`, which has useful functions like interactive login, creating a boto3 session for specific a account and role, and the programmatic versions of the `lookup` functions in `aws-sso-util`. See the documentation [here](lib/README.md).

`aws-sso-util` supersedes `aws-sso-credential-process`, which is still available in its original form [here](https://github.com/benkehoe/aws-sso-credential-process).
Read the updated docs for `aws-sso-util credential-process` [here](docs/credential-process.md).

## Quickstart

0. It's a good idea to [install the AWS CLI v2](https://docs.aws.amazon.com/cli/latest/userguide/install-cliv2.html) (which has AWS SSO support).

1. I recommend you install [`pipx`](https://pipxproject.github.io/pipx/), which installs the tool in an isolated virtualenv while linking the script you need.

Mac:
```bash
brew install pipx
pipx ensurepath
```

Other:
```bash
python3 -m pip install --user pipx
python3 -m pipx ensurepath
```

2. Install
```bash
pipx install aws-sso-util
```

3. Learn
```bash
aws-sso-util --help
```

4. Autocomplete

`aws-sso-util` uses [click](https://click.palletsprojects.com/en/7.x/), which supports autocompletion.
The details of enabling shell completion with click vary by shell ([instructions here](https://click.palletsprojects.com/en/7.x/bashcomplete/)), but here is an example that updates the completion in the background.

```bash
AWS_SSO_UTIL_COMPLETE_SCRIPT_DIR=~/.local/share/aws-sso-util
AWS_SSO_UTIL_COMPLETE_SCRIPT=$AWS_SSO_UTIL_COMPLETE_SCRIPT_DIR/complete.sh
if which aws-sso-util > /dev/null; then
  mkdir -p $AWS_SSO_UTIL_COMPLETE_SCRIPT_DIR
  (_AWS_SSO_UTIL_COMPLETE=source_bash aws-sso-util > $AWS_SSO_UTIL_COMPLETE_SCRIPT.tmp ;
    mv $AWS_SSO_UTIL_COMPLETE_SCRIPT.tmp $AWS_SSO_UTIL_COMPLETE_SCRIPT &)
  if [ -f $AWS_SSO_UTIL_COMPLETE_SCRIPT ]; then
    source $AWS_SSO_UTIL_COMPLETE_SCRIPT
  fi
fi
```

## Configuring `.aws/config`

Read the full docs for `aws-sso-util configure` and `aws-sso-util roles` [here](docs/configure.md).

You can view the roles you have available to you with `aws-sso-util roles`, which you can use to configure your profiles, but `aws-sso-util` also provides functionality to directly configure profiles for you.

`aws-sso-util configure` has two subcommands, `aws-sso-util configure profile` for configuring a single profile, and `aws-sso-util configure populate` to add _all_ your permissions as profiles, in whatever region(s) you want (with highly configurable profile names).

You probably want to set the environment variables `AWS_DEFAULT_SSO_START_URL` and `AWS_DEFAULT_SSO_REGION`, which will inform these commands of your start url and SSO region (that is, the region that you've configured AWS SSO in), so that you don't have to pass them in as parameters every time.

`aws-sso-util configure profile` takes a profile name and prompts you with the accounts and roles you have access to, to configure that profile.

`aws-sso-util configure populate` takes one or more regions, and generates a profile for each account+role+region combination.
The profile names are completely customizable.

## Logging in and out

Read the full docs for `aws-sso-util login` and `aws-sso-util logout` [here](docs/login.md).

A problem with [`aws sso login`](https://awscli.amazonaws.com/v2/documentation/api/latest/reference/sso/login.html) is that it's required to operate on a profile, that is, you have to tell it to log in to AWS SSO *plus some account and role.*
But the whole point of AWS SSO is that you log in once for *many* accounts and roles.
You could have a particular account and role set up in your default profile, but I prefer not to have a default profile so that I'm always explicitly selecting a profile and never accidentally end up in the default by mistake.
`aws-sso-util login` solves this problem by letting you *just log in* without having to think about where you'll be using those credentials.

## Adding AWS SSO support to AWS SDKs

Read the full docs for `aws-sso-util credential-process` [here](docs/credential-process.md).

Not all AWS SDKs have support for AWS SSO (which will change eventually).
However, they all have support for `credential_process`, which allows an external process to provide credentials.
`aws-sso-util credential-process` uses this to allow these SDKs to get credentials from AWS SSO.
It's added automatically (by default) by the `aws-sso-util configure` commands.

## Looking up identifiers and assignments

Read the full docs for `aws-sso-util lookup` and `aws-sso-util assignments` [here](docs/lookup.md).

When you're creating assignments through the API or CloudFormation, you're required to use identifiers like the instance ARN, the principal ID, etc.
These identifiers aren't readily available through the console, and the principal IDs are not the IDs you're familiar with.
`aws-sso-util lookup` allows you to get these identifers, even en masse.

There is no simple API for retrieving all assignments or even a decent subset.
The current best you can do is [list all the users with a particular PermissionSet on a particular account](https://docs.aws.amazon.com/singlesignon/latest/APIReference/API_ListAccountAssignments.html).
`aws-sso-util assigments` takes the effort out of looping over the necessary APIs.

## CloudFormation support

> :warning: The CloudFormation functionality is experimental. I have found the `AWS::SSO::Assignment` resources to be a little brittle and I am not yet confident they will work well if a particular assignment gets moved between nested stacks. This warning will be removed once I have confidence in the functionality.

You'll want to read the full docs [here](docs/cloudformation.md).

AWS SSO's CloudFormation support currently only includes [`AWS::SSO::Assignment`](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-sso-assignment.html), which means for every combination of principal (group or user), permission set, and target (AWS account), you need a separate CloudFormation resource.
Additionally, AWS SSO does not support OUs as targets, so you need to specify every account separately.

Obviously, this gets verbose, and even an organization of moderate size is likely to have tens of thousands of assignments.
`aws-sso-util cfn` provides two mechanisms to make this concise.

I look forward to discarding this part of the tool once there are two prerequisites:
1. OUs as targets for assignments
2. An `AWS::SSO::AssignmentGroup` resource that allows specifications of multiple principals, permission sets, and targets, and performs the combinatorics directly.

### CloudFormation Macro
`aws-sso-util` defines a resource format for an AssignmentGroup that is a combination of multiple principals, permission sets, and targets, and provides a CloudFormation Macro you can deploy that lets you use this resource in your templates.

### Client-side generation

I am against client-side generation of CloudFormation templates, but if you don't want to trust this 3rd party macro, you can generate the CloudFormation templates directly.

`aws-sso-util cfn` takes one or more input files, and for each input file, generates a CloudFormation template and potentially one or more child templates.
These templates can then be packaged and uploaded using [`aws cloudformation package`](https://awscli.amazonaws.com/v2/documentation/api/latest/reference/cloudformation/package.html) or [the SAM CLI](https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/serverless-sam-cli-install.html), for example.

The input files can either be templates using the Macro (using the `--macro` flag), or somewhat simpler configuration files using a different syntax.
These configuration files can define permission sets inline, have references that turn into template parameters, and you can provide a base template that the resulting resources are layered on top of.
