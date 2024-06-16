---
title: "Automating Terraform and Provider Updates"
date: 2024-06-15 21:00:00 +0200 
categories: [Terraform]
tags: [automation, terraform, azure pipelines, iac, azure devops, renovate]
img_path: /terraform/
---

As infrastructure as code (IaC) becomes increasingly popular, managing Terraform versions and provider dependencies efficiently is crucial. Discover in this blog post how [Renovate](https://docs.renovatebot.com/) simplifies managing Terraform versions and providers in your Git repository. Renovate allows some very specialized and complex setups, as you can configure a lot and it supports a lot of tools, but I want to show you how easy it is to set it up for Terraform.

In order for Renovate to manage your Terraform and Provider versions inside a Azure DevOps repository, you need these two things:
- Pipeline that runs Renovate on a schedule.
- A Renovate config file which tells Renovate what to do.

## Pipeline

First you create the Azure pipeline, which is actually very simple. It will run every day at 03:00 and just set up the git user mail and name and then run Renovate with some environment variables:

```yaml
schedules:
  - cron: '0 3 * * *'
    displayName: 'Every day at 3am (UTC)'
    branches:
      include: [main]
    always: true

trigger: none

pool:
  vmImage: ubuntu-latest

steps:
  - bash: |
      git config --global user.email 'bot@renovateapp.com'
      git config --global user.name 'Renovate Bot'
      npx renovate
    env:
      RENOVATE_PLATFORM: azure
      RENOVATE_ENDPOINT: $(System.CollectionUri)
      RENOVATE_TOKEN: $(System.AccessToken)
      GITHUB_COM_TOKEN: $(GITHUB_TOKEN)
      RENOVATE_REPOSITORIES: "['PROJECT_NAME/REPO_NAME']"
      LOG_LEVEL: DEBUG
```

The only two things you need to do is to reference one or more projects and repositories in which you want to run Renovate with the `RENOVATE_REPOSITORIES` env variable, and create a GitHub personal access token (read-only) that will be used to [fetch changelogs for repositories](https://docs.renovatebot.com/getting-started/running/#githubcom-token-for-changelogs). 
I stored this token as a secret pipeline variable:
![Github Token Secret Variable](renovate_github_token.png)

> Make sure the Azure DevOps build service called `PROJECT_NAME Build Service (ORGANISATION_NAME)` has permission to contribute to the specified repositories.
{: .prompt-info}

> Although it is not required, for troubleshooting at the beginning of your setup, it makes sence to turn on logs with `LOG_LEVEL: DEBUG`
{: .prompt-tip }

## Renovate Config

Once you have created the pipeline, you need to create a configuration file called `renovate.json`{: .filepath} and put it in the root of each of your repositories that you want to manage with Renovate. The filename, format, and configuration may be different for each repo - see [Renovate Configuration Options](https://docs.renovatebot.com/configuration-options/) for more information. Inside this file you can configure a lot of things, but in this guide I'll focus on Terraform. Therefore, a basic setup would look like this
```json
{
  "$schema": "https://docs.renovatebot.com/renovate-schema.json",
  "extends": [
    "config:recommended"
  ],
  "enabledManagers": [
    "terraform"
  ]  
}
```
{: file="renovate.json" }

It uses the recommended configuration and enables the Terraform Manager which, according to the [Renovate Manager](https://docs.renovatebot.com/modules/manager/) docs, includes automatic updates for these dependencies:
- terraform
- terraform-version
- terragrunt
- terragrunt-version
- tflint-plugin

And there you go. Renovate will check for new Terraform and provider versions and automatically create a pull request for you to approve:
![Example PR from Renovate](renovate_pr.png)

> Of course, it is recommended to run a validation on such a PR to check for breaking changes in your infrastructure before merging it into main.
{: .prompt-tip }

Before I come to the end, I want to show another scenario where some custom configuration was required. I had the problem that in my pipeline which is running Terraform, the Terraform version was specified as a parameter. In such a case, Renovate does not know that this parameter represents a Terraform version and therefore will not update it. BUT you can tell Renovate to treat it as a Terraform version. First, let us look at the parameters of the mentioned pipeline:
```yaml
parameters:
- name: Environment
  type: string
  default: test
  values:
  - test
  - prod
- name: TerraformVersion
  type: string
  default: 1.8.5
- name: TerraformArguments
  type: string
  default: " "
```
{: file="azure-pipelines.yml" }

As you can see, the Terraform version is specified as a default value of the `TerraformVersion` parameter. With a `customManager` entry in the `renovate.json`{: .filepath} and some regex you can tell Renovate in which file, at which position, to update the Terraform version:
```json
{
  "$schema": "https://docs.renovatebot.com/renovate-schema.json",
  "extends": [
    "config:recommended"
  ],
  "enabledManagers": [
    "terraform",
    "custom.regex"
  ],
  "customManagers": [
    {
      "customType": "regex",
      "fileMatch": [
        "^azure-pipelines\\.?[a-z0-9_.-]*\\.yml$"
      ],
      "matchStrings": [
        "- name: TerraformVersion\\n  type: string\\n  default: (?<currentValue>.*)"
      ],
      "depNameTemplate": "hashicorp/terraform",
      "depTypeTemplate": "required_version",
      "datasourceTemplate": "github-releases",
      "versioningTemplate": "hashicorp"
    }
  ]
}
```
{: file="renovate.json" }

## Conclusion
In summary, Renovate bridges the gap between complexity and simplicity, as it is very easy to use, but can be very complex depending on your needs. It makes Terraform version updates a manageable and efficient process.