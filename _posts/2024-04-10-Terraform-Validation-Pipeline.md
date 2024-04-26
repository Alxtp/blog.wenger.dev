---
title: "Mastering Terraform Validation Pipelines"
date: 2024-04-09 21:00:00 +0200 
categories: [Terraform]
tags: [automation, terraform, azure pipelines, iac, devops]
img_path: /terraform/
image: preview_yaml.jpg
---

In this blog post I want to show you how to create a Azure Pipeline which can be used to automatically validate your Terraform code and make suggestion on security, quality and best practices. How? with Static Code Analysis tools! There are a lot out there for Terraform but I picked only a few which have made the best impression. For a great comparison of these tools checkout this [post](https://devdosvid.blog/2024/04/16/a-deep-dive-into-terraform-static-code-analysis-tools-features-and-comparisons/).

I will use the following tools: 
- Trivy
- TFLint
- Semgrep OSS
- Checkov

## Pipeline
Now coming to the pipeline. Firs we need one Parameter to pass the workind directory where the Terraform root file is located. The default value `$(Build.Repository.LocalPath)` represents the local path on the agent where your source code files are downloaded. Also we define a stage for the validation:
```yaml
parameters:
- name: workingDirectory
  type: string
  default: $(Build.Repository.LocalPath)

stages:
- stage: TerraformValidationStage
  displayName: 'Terraform Validation'
```

Next you can add each tool as a different step. Note that the execution differs from tool to tool. For Trivy there is a predefined task available in the Marketplace. For the others we use bash scripts to install and run the tool.
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

To make sure that the pipeline fails only on critical findings or errors I had to run TFLint with `--minimum-failure-severity=error` and Checkov with `--hard-fail-on CRITICAL`.
