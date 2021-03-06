# SSO Sync

> Helping you populate AWS SSO directly with your Google Apps users

SSO Sync will run on any platform that Go can build for.

## Why?

As per the [AWS SSO](https://aws.amazon.com/single-sign-on/) Homepage:

> AWS Single Sign-On (SSO) makes it easy to centrally manage access
> to multiple AWS accounts and business applications and provide users
> with single sign-on access to all their assigned accounts and applications
> from one place.

Key part further down:

> With AWS SSO, you can create and manage user identities in AWS SSO’s
>identity store, or easily connect to your existing identity source including
> Microsoft Active Directory and **Azure Active Directory (Azure AD)**.

AWS SSO can use other Identity Providers as well... such as Google Apps for Domains. Although AWS SSO
supports a subset of the SCIM protocol for populating users, it currently only has support for Azure AD.

This project provides a CLI tool to pull users and groups from Google and push them into AWS SSO.
`ssosync` deals with removing users as well. The heavily commented code provides you with the detail of
what it is going to do.

### References

 * [SCIM Protocol RFC](https://tools.ietf.org/html/rfc7644)
 * [AWS SSO - Connect to Your External Identity Provider](https://docs.aws.amazon.com/singlesignon/latest/userguide/manage-your-identity-source-idp.html)
 * [AWS SSO - Automatic Provisioning](https://docs.aws.amazon.com/singlesignon/latest/userguide/provision-automatically.html)

## Installation

You can `go get github.com/awslabs/ssosync` or grab a Release binary from the release page. The binary
can be used from your local computer, or you can deploy to AWS Lambda to run on a CloudWatch Event
for regular synchronization.

## Configuration

You need a few items of configuration. One side from AWS, and the other
from Google Cloud / Apps to allow for API access to each. You should have configured
Google as your Identity Provider for AWS SSO already.

You will need the files produced by these steps for AWS Lambda deployment as well
as locally running the ssosync tool.

### Google

Head to the [Google Cloud Console](https://console.cloud.google.com/) for your Domain
(Specifically API & Services ->
[Credentials](https://console.cloud.google.com/projectselector2/apis/credentials))
and Create a Project.

Creating a project will take a few seconds. Once it is complete, you can then Configure the Consent
Screen (there will be a clear warning and button for it). Click Through and select "Internal". Give
a name and press Save as you don't need the rest.

Now go back to Credentials, Click Create Credentials and then select OAuth client ID. Select Other and
provide a name. You will be displayed credentials, press okay and then use the download button, and a
JSON file will download.

**THIS FILE IS IMPORTANT AND SECRET - KEEP IT SAFE**

With this done, you can log in and generate a token.json file. To create the file, use the
`ssosync google` command. With help output, it looks like this:

```text
Log in to Google - use me to generate the files needed for the main command

Usage:
  ssosync google [flags]

Flags:
  -h, --help               help for google
      --path string        set the path to find credentials (default "credentials.json")
      --tokenPath string   set the path to put token.json output into (default "token.json")
```

When you run the command correctly, it will give a URL to load in your browser. Go to it, and you'll get
a string to paste back and enter. Once you paste the line in, the file generates.

The Token file is useless without the Credentials File - but keep it safe.

Back in the Console go to the Dashboard for the API & Services and select "Enable API and Services".
In the Search box type `Admin` and select the `Admin SDK` option. Click the `Enable` button.

### AWS

Go to the AWS Single Sign-On console in the region you have set up AWS SSO and select
Settings. Click `Enable automatic provisioning`.

A pop up will appear with URL and the Access Token. The Access Token will only appear
at this stage. You want to copy both of these into a text file which ends in the extension
`.toml`.

```toml
Token    = "tokenHere"
Endpoint = "https://scim.eu-west-1.amazonaws.com/a-guid-would-be-here/scim/v2/"
```

## Local Usage

Usage:

The default for ssosync is to run through the sync.

```text
A command line tool to enable you to synchronise your Google
Apps (G-Suite) users to AWS Single Sign-on (AWS SSO)

Usage:
  ssosync [flags]
  ssosync [command]

Available Commands:
  google      Log in to Google
  help        Help about any command

Flags:
  -d, --debug                          Enable verbose / debug logging
  -c, --googleCredentialsPath string   set the path to find credentials for Google (default "credentials.json")
  -t, --googleTokenPath string         set the path to find token for Google (default "token.json")
  -h, --help                           help for ssosync
  -s, --scimConfig string              AWS SSO SCIM Configuration (default "aws.toml")

Use "ssosync [command] --help" for more information about a command.
```

The output of the command when run without 'debug' turned on looks like this:

```
2020-05-26T12:08:14.083+0100	INFO	cmd/root.go:43	Creating the Google and AWS Clients needed
2020-05-26T12:08:14.084+0100	INFO	internal/sync.go:38	Start user sync
2020-05-26T12:08:14.979+0100	INFO	internal/sync.go:73	Clean up AWS Users
2020-05-26T12:08:14.979+0100	INFO	internal/sync.go:89	Start group sync
2020-05-26T12:08:15.578+0100	INFO	internal/sync.go:135	Start group user sync	{"group": "AWS Administrators"}
2020-05-26T12:08:15.703+0100	INFO	internal/sync.go:172	Clean up AWS groups
2020-05-26T12:08:15.703+0100	INFO	internal/sync.go:183	Done sync groups
```

## AWS Lambda Usage

NOTE: Using Lambda may incur costs in your AWS account. Please make sure you have checked
the pricing for AWS Lambda and CloudWatch before continuing.

Running ssosync once means that any changes to your Google directory will not appear in
AWS SSO. To sync. regularly, you can run ssosync via AWS Lambda. 

You will find using the provided CDK deployment scripts the easiest method. Install
the [AWS CDK](https://aws.amazon.com/cdk/) before you start.

### Using the right binary for AWS Lambda

You require the AMD64 binary for AWS Lambda. This can be either downloaded from the
Releases page, or built locally. A great way to do this to use
[goreleaser](https://goreleaser.com/) in Snapshot mode which will build the various
system binaries.

Whichever route you take, the CDK stack for deployment requires a folder which only
contains the binary and nothing else. goreleaser will take care of this for you; just
be aware if you are obtaining a binary from any other route.

NOTE: The binaries tagged v0.0.1 on GitHub are not suitable for AWS Lambda usage.

To build with goreleaser you can expect the following kind of output:

```
$ goreleaser build --snapshot

   • building...
   • loading config file       file=.goreleaser.yml
   • running before hooks
      • running go mod download
   • loading environment variables
   • getting and validating git state
      • releasing v0.0.1, commit fcc9977a10ae24a92417b00472267ec9bc40aada
      • pipe skipped              error=disabled during snapshot mode
   • parsing tag
   • setting defaults
      • snapshotting
      • github/gitlab/gitea releases
      • project name
      • building binaries
      • creating source archive
      • archives
      • linux packages
      • snapcraft packages
      • calculating checksums
      • signing artifacts
      • docker images
      • artifactory
      • blobs
      • homebrew tap formula
      • scoop manifests
   • snapshotting
   • checking ./dist
   • writing effective config file
      • writing                   config=dist/config.yaml
   • generating changelog
      • pipe skipped              error=not available for snapshots
   • building binaries
      • building                  binary=/Users/leepac/go/src/github.com/awslabs/ssosync/dist/ssosync_windows_amd64/ssosync.exe
      • building                  binary=/Users/leepac/go/src/github.com/awslabs/ssosync/dist/ssosync_linux_arm64/ssosync
      • building                  binary=/Users/leepac/go/src/github.com/awslabs/ssosync/dist/ssosync_linux_386/ssosync
      • building                  binary=/Users/leepac/go/src/github.com/awslabs/ssosync/dist/ssosync_linux_arm_6/ssosync
      • building                  binary=/Users/leepac/go/src/github.com/awslabs/ssosync/dist/ssosync_linux_amd64/ssosync
      • building                  binary=/Users/leepac/go/src/github.com/awslabs/ssosync/dist/ssosync_darwin_amd64/ssosync
   • build succeeded after 7.31s
```

### Deploying using the AWS CDK

You need to know the locations of the credentials.json, token.json and aws.toml files
that you used for the configuration of ssosync. You also need the binary folder location.

With these files in hand, head into the `deployments/cdk` folder and then run the cdk
deploy command with the AWS_TOML, GOOGLE_CREDENTIALS, GOOGLE_TOKEN and SSOSYNC_PATH
variables set:

NOTE: You might get a warning showing you need to execute `cdk bootstrap` if you have
never used the AWS CDK in the account/region before. You can just run that command
beforehand to solve this.

#### *nix

```
AWS_TOML=../../aws.toml GOOGLE_CREDENTIALS=../../credentials.json GOOGLE_TOKEN=../../token.json SSOSYNC_PATH=../../dist/ssosync_linux_amd64 cdk deploy
```

#### Windows (PowerShell)

```
$env:AWS_TOML = '../../aws.toml'
$env:GOOGLE_CREDENTIALS = '../../credentials.json'
$env:GOOGLE_TOKEN = '../../token.json'
$env:SSOSYNC_PATH = '../../dist/ssosync_linux_amd64'
cdk deploy
```

```
$ AWS_TOML=../../aws.toml GOOGLE_CREDENTIALS=../../credentials.json GOOGLE_TOKEN=../../token.json SSOSYNC_PATH=../../dist/ssosync_linux_amd64 cdk deploy
  This deployment will make potentially sensitive changes according to your current security approval level (--require-approval broadening).
  Please confirm you intend to make the following modifications:
  
  IAM Statement Changes
  ┌───┬──────────────────────────────────┬────────┬──────────────────────────────────┬──────────────────────────────────┬────────────────────────────────────┐
  │   │ Resource                         │ Effect │ Action                           │ Principal                        │ Condition                          │
  ├───┼──────────────────────────────────┼────────┼──────────────────────────────────┼──────────────────────────────────┼────────────────────────────────────┤
  │ + │ ${AwsToml}                       │ Allow  │ secretsmanager:GetSecretValue    │ AWS:${SsoSync/ServiceRole}       │                                    │
  │   │ ${GoogleCred}                    │        │                                  │                                  │                                    │
  │   │ ${GoogleToken}                   │        │                                  │                                  │                                    │
  ├───┼──────────────────────────────────┼────────┼──────────────────────────────────┼──────────────────────────────────┼────────────────────────────────────┤
  │ + │ ${SsoSync.Arn}                   │ Allow  │ lambda:InvokeFunction            │ Service:events.amazonaws.com     │ "ArnLike": {                       │
  │   │                                  │        │                                  │                                  │   "AWS:SourceArn": "${Rule.Arn}"   │
  │   │                                  │        │                                  │                                  │ }                                  │
  ├───┼──────────────────────────────────┼────────┼──────────────────────────────────┼──────────────────────────────────┼────────────────────────────────────┤
  │ + │ ${SsoSync/ServiceRole.Arn}       │ Allow  │ sts:AssumeRole                   │ Service:lambda.amazonaws.com     │                                    │
  └───┴──────────────────────────────────┴────────┴──────────────────────────────────┴──────────────────────────────────┴────────────────────────────────────┘
  IAM Policy Changes
  ┌───┬────────────────────────┬────────────────────────────────────────────────────────────────────────────────┐
  │   │ Resource               │ Managed Policy ARN                                                             │
  ├───┼────────────────────────┼────────────────────────────────────────────────────────────────────────────────┤
  │ + │ ${SsoSync/ServiceRole} │ arn:${AWS::Partition}:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole │
  └───┴────────────────────────┴────────────────────────────────────────────────────────────────────────────────┘
  (NOTE: There may be security-related changes not in this list. See https://github.com/aws/aws-cdk/issues/1299)
  
  Do you wish to deploy these changes (y/n)? y
  SsoSyncStack: deploying...
  [0%] start: Publishing d5e2919f38e8204910b42413d033318f5f422a8489e3bbe706bb21458622971e:current
  [100%] success: Published d5e2919f38e8204910b42413d033318f5f422a8489e3bbe706bb21458622971e:current
  SsoSyncStack: creating CloudFormation changeset...
   0/10 | 13:22:57 | CREATE_IN_PROGRESS   | AWS::SecretsManager::Secret | AwsToml
   0/10 | 13:22:57 | CREATE_IN_PROGRESS   | AWS::CDK::Metadata          | CDKMetadata
   0/10 | 13:22:57 | CREATE_IN_PROGRESS   | AWS::SecretsManager::Secret | GoogleCred
   0/10 | 13:22:57 | CREATE_IN_PROGRESS   | AWS::SecretsManager::Secret | GoogleToken
   0/10 | 13:22:57 | CREATE_IN_PROGRESS   | AWS::IAM::Role              | SsoSync/ServiceRole (SsoSyncServiceRoleE85B4FFE)
   0/10 | 13:22:57 | CREATE_IN_PROGRESS   | AWS::IAM::Role              | SsoSync/ServiceRole (SsoSyncServiceRoleE85B4FFE) Resource creation Initiated
   0/10 | 13:22:59 | CREATE_IN_PROGRESS   | AWS::SecretsManager::Secret | GoogleCred Resource creation Initiated
   0/10 | 13:22:59 | CREATE_IN_PROGRESS   | AWS::SecretsManager::Secret | AwsToml Resource creation Initiated
   1/10 | 13:22:59 | CREATE_COMPLETE      | AWS::SecretsManager::Secret | GoogleCred
   1/10 | 13:22:59 | CREATE_IN_PROGRESS   | AWS::SecretsManager::Secret | GoogleToken Resource creation Initiated
   2/10 | 13:22:59 | CREATE_COMPLETE      | AWS::SecretsManager::Secret | AwsToml
   2/10 | 13:22:59 | CREATE_IN_PROGRESS   | AWS::CDK::Metadata          | CDKMetadata Resource creation Initiated
   3/10 | 13:22:59 | CREATE_COMPLETE      | AWS::SecretsManager::Secret | GoogleToken
   4/10 | 13:22:59 | CREATE_COMPLETE      | AWS::CDK::Metadata          | CDKMetadata
   5/10 | 13:23:11 | CREATE_COMPLETE      | AWS::IAM::Role              | SsoSync/ServiceRole (SsoSyncServiceRoleE85B4FFE)
   5/10 | 13:23:13 | CREATE_IN_PROGRESS   | AWS::IAM::Policy            | SsoSync/ServiceRole/DefaultPolicy (SsoSyncServiceRoleDefaultPolicy1A9D4C1C)
   5/10 | 13:23:14 | CREATE_IN_PROGRESS   | AWS::IAM::Policy            | SsoSync/ServiceRole/DefaultPolicy (SsoSyncServiceRoleDefaultPolicy1A9D4C1C) Resource creation Initiated
   6/10 | 13:23:27 | CREATE_COMPLETE      | AWS::IAM::Policy            | SsoSync/ServiceRole/DefaultPolicy (SsoSyncServiceRoleDefaultPolicy1A9D4C1C)
   6/10 | 13:23:30 | CREATE_IN_PROGRESS   | AWS::Lambda::Function       | SsoSync (SsoSync48C335B6)
   6/10 | 13:23:31 | CREATE_IN_PROGRESS   | AWS::Lambda::Function       | SsoSync (SsoSync48C335B6) Resource creation Initiated
   7/10 | 13:23:31 | CREATE_COMPLETE      | AWS::Lambda::Function       | SsoSync (SsoSync48C335B6)
   7/10 | 13:23:34 | CREATE_IN_PROGRESS   | AWS::Events::Rule           | Rule (Rule4C995B7F)
   7/10 | 13:23:34 | CREATE_IN_PROGRESS   | AWS::Events::Rule           | Rule (Rule4C995B7F) Resource creation Initiated
  7/10 Currently in progress: Rule4C995B7F
   8/10 | 13:24:35 | CREATE_COMPLETE      | AWS::Events::Rule           | Rule (Rule4C995B7F)
   8/10 | 13:24:37 | CREATE_IN_PROGRESS   | AWS::Lambda::Permission     | SsoSync/AllowEventRuleSsoSyncStackRule051D4243 (SsoSyncAllowEventRuleSsoSyncStackRule051D4243FDBD7EFC)
   8/10 | 13:24:38 | CREATE_IN_PROGRESS   | AWS::Lambda::Permission     | SsoSync/AllowEventRuleSsoSyncStackRule051D4243 (SsoSyncAllowEventRuleSsoSyncStackRule051D4243FDBD7EFC) Resource creation Initiated
  
   ✅  SsoSyncStack
  
  Stack ARN:
  arn:aws:cloudformation:us-east-1:xxxx:stack/SsoSyncStack/b2297840-xxxx-xxxx-xxxx-0ea20f614b35
```
