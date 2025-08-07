---
title: "Create Terraform Validation Pipelines"
date: 2024-04-27 21:00:00 +0200 
categories: [Terraform]
tags: [automation, terraform, azure pipelines, iac, devops, static code analysis, trivy, tflint, semgrep oss, checkov]
media_subpath: /terraform/
---

In this blog post, I want to show you how to create an Azure pipeline that can be used to automatically validate your Terraform code and make suggestions for security, quality and best practices so that you can improve your scrappy code. How? using static code analysis tools! There are a lot out there for Terraform and the major private cloud providers, but I have picked just a few that have made the best impression. For a great comparison of these tools, check out this [post](https://devdosvid.blog/2024/04/16/a-deep-dive-into-terraform-static-code-analysis-tools-features-and-comparisons/).

I will use the following tools: 
- Trivy
- TFLint
- Semgrep OSS
- Checkov

## Pipeline

Now for the pipeline. First, we need a parameter to pass the working directory where the Terraform root file is located. The default value `$(Build.Repository.LocalPath)` represents the local path on the agent where your source files will be downloaded. We also define a stage for validation:

```yaml
parameters:
- name: workingDirectory
  type: string
  default: $(Build.Repository.LocalPath)

stages:
- stage: TerraformValidationStage
  displayName: 'Terraform Validation'
```

Next you can add each tool as a seperate step. Note that the execution differs from tool to tool. For Trivy there is a predefined task available in the marketplace. For the others we use bash scripts to install and run the tool.

```yaml
parameters:
- name: workingDirectory
  type: string
  default: $(Build.Repository.LocalPath)

jobs:
- job: TerraformValidation
  displayName: 'Terraform Validate'
  steps:
  - checkout: self 
  
  - task: trivy@1
    displayName: 'Trivy'
    inputs:
    version: '0.50.1'
    path: ${{ parameters.workingDirectory }}
    exitCode: '0'
  
  - script: |
      curl -s https://raw.githubusercontent.com/terraform-linters/tflint/master/install_linux.sh | bash

      tflint --version
      tflint --init
      tflint --minimum-failure-severity=error
    workingDirectory: ${{ parameters.workingDirectory }}
    displayName: 'TFLint'
  
  - script: |
      python -m pip install --upgrade pip
      pip install semgrep

      semgrep --version
      semgrep scan --config auto
    workingDirectory: ${{ parameters.workingDirectory }}
    displayName: 'Semgrep OSS'
  
  - script: |
      apt install python3-pip
      pip3 install checkov

      checkov --version
      checkov -d . --hard-fail-on CRITICAL
    workingDirectory: ${{ parameters.workingDirectory }}
    displayName: 'Checkov'
```

For TFLint you need to add a file called `.tflint.hcl`{: .filepath} in the root of your directory which contains the providers you want to validate. In my case I use `azurerm`.
```hcl
plugin "azurerm" {
  enabled = true
  version = "0.26.0"
  source  = "github.com/terraform-linters/tflint-ruleset-azurerm"
}
```
{: file=".tflint.hcl" }

> To make sure that the pipeline fails only on critical findings or errors, I ran TFLint with `--minimum-failure-severity=error` and Checkov with `--hard-fail-on CRITICAL`.
{: .prompt-info }

And there you go, you have built yourself a pipeline with the major Terraform static code analyzers. If you need all of them, I can't answer that, but maybe I can make the decision easier with the following sample run to compare the tools.

## Example Run

Following I will show you the pipeline in action with a simple terraform setup with a few obvious mistakes to check what each tool has to say.

```hcl
locals {
  var_with_no_use = "This is a variable with no use"
}

resource "azurerm_resource_group" "example" {
  name     = "example-resources"
  location = "switzerlandnorth"
}

resource "azurerm_storage_account" "example" {
  name                      = "storageaccountname"
  resource_group_name       = azurerm_resource_group.example.name
  location                  = azurerm_resource_group.example.location
  account_tier              = "Standard"
  account_replication_type  = "LRS"
  min_tls_version           = "TLS1_0" # Outdated tls version
  enable_https_traffic_only = false # http enabled
}
```
{: file="main.tf" }

### Trivy

Trivy, which doesn't output to the console but to its own "trivy" tab next to the pipeline run summary, warned me about the deprecated tls version and suggested that I enable `Secure transfer required`, which is recommended by Microsoft:

![Trivy Example Validation](trivy.png)
_Trivy Example Validation_

### TFLint

TFLint just informed me about the unused variable:

![TFLint Example Validation](tflint.png)
_TFLint Example Validation_

### Semgrep OSS

With semgrep we have received again a different suggestion. Firstly, the outdated TLS version again, then that http traffic should be deactivated and also that the storage analytics logs should be used:

![Semgrep OSS Example Validation](semgrep.png)
_Semgrep OSS Example Validation_

### Checkov

We come to the last and most detailed tool, checkov. With 12 failed checks, I can't even show them in one picture, so below is a list of the things found.

![Checkov Example Validation](checkov.png)
_Checkov  Example Validation_

- Ensure that Storage accounts disallow public access
- Ensure Storage Account is using the latest version of TLS encryption
- Ensure that Storage blobs restrict public access
- Ensure that 'enable_https_traffic_only' is enabled
- Ensure that Storage Accounts use replication
- Ensure Storage logging is enabled for Queue service for read, write and delete requests
- Ensure storage account is configured with SAS expiration policy
- Ensure storage account is configured with private endpoint
- Ensure soft-delete is enabled on Azure storage account
- Ensure storage account is configured without blob anonymous access
- Ensure storage account is not configured with Shared Key authorization
- Ensure storage for critical data are encrypted with Customer Managed Key

## Conclusion

As you can see, all the tools vary HEAVILY on the suggestions; some are very general and focus more on syntax and unused declarations, and others more on security aspects and best practices of the resources themselves. You have to decide which one to use based on your needs. You may want to use all of them together, as each tool will have its own suggestions that another tool may not have mentioned, even if things are brought up repeatedly.
