+++
title = "Migrating to Terraform Cloud and managing workspaces in a simpler way"
description = "My journey to migrate to Terraform Cloud and attempt to make workspaces management easier"
tags = [
  "automation",
  "cloud",
  "devops",
  "enterprise",
  "hashicorp",
  "infrastructure-as-code",
  "terraform",
  "terraformcloud",
]
date = "2020-04-14"
categories = [
  "automation",
  "cloud",
  "devops",
  "enterprise",
  "hashicorp",
  "infrastructure-as-code",
  "terraform",
  "terraformcloud",
]
highlight = "true"
+++

## TL;DR

This writing is an attempt to highlight the challenges encountered when I migrated to Terraform Cloud. I tried to condense the key parts of the journey which I felt relevant to mention.

I will talk about an open-source command-line tool which I wrote: [Terraform Cloud Wrapper](https://github.com/mvisonneau/tfcw) (TFCW). This tool aims to reduce the amount of glue required in order to operate it. I reckon that the edges are quite opinionated but I believe it could be interesting to see some of its featureset included with a more generic approach into the upstream projects.

## Timeline

In **May 2019**, [Hashicorp](https://en.wikipedia.org/wiki/HashiCorp) announced they were opening part of their (formerly known as) Terraform Enterprise offering to everyone for FREE!

{{< tweet 1129115275918487555 >}}

For the fervent open-source users of the tool, the main outcome was that they would not have to worry about maintaining their [remote backends for state storage and locking](https://www.terraform.io/docs/backends/state.html) on their own anymore.

A few months later, **in September 2019**. Hashicorp [announced a new featureset](https://www.hashicorp.com/blog/announcing-terraform-cloud/) for Terraform Cloud, also available for free. This included several functionnalities around **collaboration** and **automation** that users would often have to workout themselves beforehand.

## Usecase

In our case, we started this journey earlier this year with the migration an existing Terraform estate composed of **~50 projects** with a total of **~250 states**.

Our implementation was already following [Hashicorp's recommended practices for Terraform](https://www.terraform.io/docs/cloud/guides/recommended-practices/index.html). All projects were running with versions of Terraform `0.12.x` excepted 3 which were still on `0.11.4`.

We were using a [S3 bucket](https://docs.aws.amazon.com/AmazonS3/latest/dev/UsingBucket.html) as the remote state storage with a [DynamoDB table](https://aws.amazon.com/dynamodb/features/) as the locking mechanism.

The main purpose of this Terraform codebase is to manage `AWS` resources. Although we are also using it with another dozen of providers such as `Vault`, `Cloudflare`, `Datadog`, `GCP` and a few others like `template` or `archive`.

## Getting started

We did our migration in 2 phases:

- [First, migrating the remote states](#first-migrating-the-remote-states)
- [Then, leveraging remote plans & applies](#then-leveraging-remote-plans--applies)

As we could expect, a very complete [terraform provider to configure Terraform Cloud](https://www.terraform.io/docs/providers/tfe/index.html) is available. This is what we got started with in order to manage our workspaces. At first our motivation was to manage their creation and to ensure they were configured to not run remote plans & applies (which is the default when using a TFC workspace as a remote backend).

```hcl
resource "tfe_workspace" "foo" {
  name         = "foo"
  organization = "acme"

  # Disable remote plans & applies
  operations   = false
}
```

### First, migrating the remote states

Moving from `S3+DynamoDB` to `Terraform Cloud` was a very smooth process. It went as easy as replacing the remote block configuration:

> before

```hcl
terraform {
  backend "s3" {
    bucket         = "foo"
    key            = "path/to/my/key"
    region         = "eu-west-1"
    dynamodb_table = "bar"
  }
}
```

> after

```hcl
terraform {
  backend "remote" {
    hostname     = "app.terraform.io"
    organization = "acme"

    workspaces {
      name = "foo"
    }
  }
}
```

The other thing that needed to be graduately replaced were the [terraform_remote_state](https://www.terraform.io/docs/providers/terraform/d/remote_state.html) data sources:

> before

```hcl
data "terraform_remote_state" "foo" {
  backend = "s3"

  config {
    bucket = "foo"
    key    = "path/to/my/key"
    region = "eu-west-1"
  }
}
```

> after

```hcl
data "terraform_remote_state" "foo" {
  backend = "remote"

  config = {
    hostname     = "app.terraform.io"
    organization = "acme"

    workspaces = {
      name = "foo"
    }
  }
}
```

We had almost no surprises during the migration process. The only issue we faced was related to a third party / open source tool that we were using: [Terraboard](https://github.com/camptocamp/terraboard).

Terraboard was initialy made to work exclusively on `S3+DynamoDB`. Thanks to [@raphink](https://github.com/raphink) for [merging my PR](https://github.com/camptocamp/terraboard/pull/67), we now all have support for `Terraform Cloud` since [v0.17.0](https://github.com/camptocamp/terraboard/releases/tag/0.17.0)!

### Then, leveraging remote plans & applies

When we started to look at this second phase, running our Terraform plans and applies remotely on Terraform Cloud. We rapidly understood that this would imply much bigger changes than the first phase. As the execution of Terraform takes place remotely, there was a few things that needed to shift from our CI perimiter to the Terraform Cloud one.

#### Modules

Loading modules which are not stored in the root directory requires some extra configuration. In our case we had 2 different scenarios:

##### Located in another repository, fetched using SSH

We also sometimes want to share our modules between repositories, in this case we need to have sufficient permissions to be able to fetch the referenced git repository. (NB: it can also be within itself for versioning purposes)

> example

```hcl
module "foo" {
  source = "git::git@git.example.com:foo/bar.git//mymodule?ref=cc4fb62"
}
```

For this scenario, Terraform Cloud supports the importation of [private SSH keys](https://www.terraform.io/docs/cloud/api/ssh-keys.html).

This gets therefore easily solved with the following configuration:

```hcl
resource "tfe_ssh_key" "git" {
  name         = "git"
  organization = "acme"

  # Of course you will not store this value in plaintext in your repo 🙏
  key          = "private-ssh-key"
}

resource "tfe_workspace" "foo" {
  name         = "foo"
  organization = "acme"
  ssh_key_id   = tfe_ssh_key.git.id
}
```

##### Located within the same repository but in another folder

In our case we standardized our directory tree pattern across our repositories. Here is an example of what they can look like:

```bash
~$ tree -d
.
├── modules
│   └── aws
│       └── foo
└── providers
    └── aws
        ├── dev
        │   ├── ap-east-1
        │   ├── eu-west-1
        │   └── us-east-1
        ├── prod
        │   ├── ap-east-1
        │   ├── eu-west-1
        │   └── us-east-1
        └── staging
            ├── ap-east-1
            ├── eu-west-1
            └── us-east-1
```

This translates to with not so great modules invocations which can look like this:

```hcl
module "foo" {
  source = "../../../../modules/aws/foo"
}
```

We therefore had to configure most of our workspaces with the following in order to let Terraform Cloud process. Terraform Cloud allows you to configure a [working directory](https://www.terraform.io/docs/cloud/workspaces/settings.html#terraform-working-directory) parameter to overcome this issue.

```hcl
resource "tfe_workspace" "foo" {
  name         = "foo"
  organization = "acme"
  
  working_directory = "providers/aws/dev/eu-west-1"
}
```

#### Credentials

So far we were able to handle things in nicely fashion, this was however about to change. I will solely focus on one usecase here though, **AWS authentication**.

There are [various ways](https://www.terraform.io/docs/providers/aws/index.html#authentication) to allow Terraform to authenticate against AWS. In our case, we chose to generate [STS tokens](https://docs.aws.amazon.com/STS/latest/APIReference/Welcome.html) with short (~15 to 30m) Time-To-Live (TTL) using [Vault AWS secret engine](https://www.vaultproject.io/docs/secrets/aws).

In our existing implementation, our CI workers were making the required API call to Vault right before triggering the Terraform run. However, we now had to do an additionnal step and updated these variables onto Terraform Cloud workspace.

It seemed quite complex and innefficient to attempt doing this using the [Vault provider](https://www.terraform.io/docs/providers/vault/index.html) and the [tfe_variable](https://www.terraform.io/docs/providers/tfe/r/variable.html) resource. We would have had to chain our Terraform runs and trigger an apply on our stack managing Terraform Cloud, wait for it to complete and then trigger our actual intended run.

We were also starting to feel bad about this glue we were already carrying from the past. This is why I decided to look into a more tailored approach and started writing [Terraform Cloud Wrapper](https://github.com/mvisonneau/tfcw) (TFCW).

The objectives were simple:

- Reduce the amount of glue & overhead required to manage dynamic variable values
- Still define as code, in a declarative fashion how to fetch those variable values

Using **TFCW**, we only need to add an additional file to our directories to sort this issue out:

```hcl
# tfcw.hcl
envvar "_" {
  vault {
    method = "write"
    path   = "aws/sts/dev"

    keys = {
      access_key     = "AWS_ACCESS_KEY_ID",
      secret_key     = "AWS_SECRET_ACCESS_KEY",
      security_token = "AWS_SESSION_TOKEN",
    }

    params = {
      ttl = "30m"
    }
  }
}
```

You would then have to run it, and automatically the variables would get updated onto your workspace accordingly

```bash
~$ tfcw render
INFO[] Checking workspace configuration
INFO[] Processing variables and updating their values on TFC
INFO[] Set variable 'AWS_ACCESS_KEY_ID' (environment)
INFO[] Set variable 'AWS_SECRET_ACCESS_KEY' (environment)
INFO[] Set variable 'AWS_SESSION_TOKEN' (environment)
```

We also figured that as Terraform runs happen remotely and are interfacable through the [Terraform Cloud API](https://www.terraform.io/docs/cloud/api/index.html). We might as well doing the end-to-end deployment in a single command:

```bash
~$ tfcw run create
INFO[] Checking workspace configuration
INFO[] Processing variables and updating their values on TFC
INFO[] Set variable 'AWS_ACCESS_KEY_ID' (environment)
INFO[] Set variable 'AWS_SECRET_ACCESS_KEY' (environment)
INFO[] Set variable 'AWS_SESSION_TOKEN' (environment)
INFO[] Preparing plan
Terraform v0.12.24
Configuring remote state backend...
Initializing Terraform configuration...
[...]
```

## Rationalizing

We were now managing our Terraform Cloud configuration from different places and we felt that this could lead to confusion, specially as the scale grows. Which is why we decided to go for a all-in-one solution that would allow us to manage all our prerequisites from one location. Hence, a classic workspace configuration now looks like this:

```hcl
tfc {
  # Here we ensure that the workspace is configured
  # as we want it, before each Terraform Cloud run
  workspace {
    operations        = true
    auto-apply        = false
    terraform-version = "0.12.24"
    working-directory = "providers/aws/prod/eu-west-1"
    ssh-key           = "gitlab"
  }
  
  purge-unmanaged-variables = true
}

# We have some defaults for our variable definitions
defaults {
  ttl = "15m"
  s5 {
    engine = "vault"
    vault {
      transit-key = "s5_rds_prod"
    }
  }
}

# A terraform variable to configure a resource
# Guick and always good reminder that this value will
# then be stored in plaintext within the statefile
tfvar "api_token" {
  ttl = "1d"
  s5 {
    value = "{{s5:LUUjOgw3Glb/lLKH4cK8G0xTP9g1PGxkrHvWA==}}"
  }
}

# We configure our STS token to let Terraform Cloud
# authenticate against AWS APIs
envvar "_" {
  vault {
    method = "write"
    path   = "aws/sts/prod"

    keys = {
      access_key     = "AWS_ACCESS_KEY_ID",
      secret_key     = "AWS_SECRET_ACCESS_KEY",
      security_token = "AWS_SESSION_TOKEN",
    }

    params = {
      ttl = "30m"
    }
  }
}
```

To see supported options, feel free to have a quick look at the [configuration syntax documentation](https://github.com/mvisonneau/tfcw/blob/master/docs/configuration_syntax.md).

This freed us from having to maintain yet another Terraform repository to manage our Terraform Cloud configuration. Repository which on top of that was not managing 100% of our desired configuration.

It also allowed us to have some command line helpers specific to Terraform cloud. Such as discarding or approving a run:

```bash
~$ tfcw run approve --current
INFO[] Approving run ID: run-wjSaRYGNuCqhfeNx
Terraform v0.12.24
Initializing plugins and modules...
[..]

~$ tfcw run discard --current
INFO[] Discarding run ID: run-yb6rQZeckXcPb2Ka
```

or easily removing the variables once our run has been applied:

```bash
~$ tfcw workspace delete-variables --all
INFO[] deleted variable AWS_ACCESS_KEY_ID
INFO[] deleted variable AWS_SECRET_ACCESS_KEY
INFO[] deleted variable AWS_SESSION_TOKEN
```

## Terraform Cloud feedback after using it for a couple of months

We are now using Terraform Cloud with remote runs for all our workspaces and are very pleased about it. Some improvements that we could wish for the future would be (1) increased parallelism capabilities in the free plan and (2) speeeed!

It is not usually that important when waiting through CI pipelines to get through but can become a really dissapointing experience when you are troubleshooting some plans locally are were used to the local experience. (to overcome this situation, I am using this [hacky function](https://github.com/mvisonneau/tfcw#perform-local-terraform-runs) in my shell profile though).

## Final thoughts

Whether you are just looking into Terraform Cloud or are already leveraging it extensively. I would recommend to evaluate how _reliable_, _maintainable_ and _scalable_ your implementation is and figure out what you can do to improve it!
