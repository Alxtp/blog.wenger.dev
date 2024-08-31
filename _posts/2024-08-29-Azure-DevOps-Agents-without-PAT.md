---
title: "Simple and Secure: Deploy self-hosted Azure DevOps Agents without PAT"
date: 2024-08-29 21:00:00 +0200
categories: [Azure]
tags: [azure, automation, terraform, iac, azure devops, container, docker]
img_path: /azure/
---

In the world of Azure DevOps, self-hosted agents provide greater control and customization for your CI/CD pipelines. However, traditional deployment methods often rely on personal access tokens (PATs) such as the [official documentation of Microsoft of deploying self-hosted agents with container app jobs](https://learn.microsoft.com/en-us/azure/container-apps/tutorial-ci-cd-runners-jobs?tabs=bash&pivots=container-apps-jobs-self-hosted-ci-cd-azure-pipelines), which can introduce security risks and are bound to a user account. 

This post explores a modern, secure approach to deploying self-hosted Azure DevOps agents using Azure Container Apps jobs and managed identity and also using Terraform to automatically create it. To also secure the nework traffic the whole setup will be compleatly vnet intgrated. This will require an Firewall or some other public endpoint, in this guide i'll use an Azure Firewall. 

## Prerequisites
- Azure Account
- Azure CLI: Install the Azure CLI.
- Terraform: Install Terraform.
- Docker: Install Docker Desktop.
- Azure DevOps organization

## Architecture
The final architecture looks like this:

![Architecture Diagramm](architecture.png)
At the core is a **Container Apps Environment** with a **Internal Workload Profile**, which includes:

- **Virtual Network and Subnet** - Private network in which the container will run
- **Private Endpoint and Private** DNS Zone - Required by the Azure Monitor Private Link Scope
- **Container App Job** - Will start up a docker container if there are jobs in the Azure DevOps pool

The Containers connect to an Azure Firewall for managing outgoing traffic.


Furthermore the architecture incorporates:

- **Container Registry** for storing and managing container images
- **User Assigned Managed Identity** for access to the Container Registry and to the Agent Pool in Azure DevOps
- **Azure Monitor Private Link Scope** for private integrration of the Log Analytics Workspace
- **Log Analytics Workspace** in order to monitor and troublshoot container runs

## Setup Terraform
To deploy the Azure resources I will use Terraform which simplfies the creation and management of all the resources. See the required providers. The state will be safed to an Azure Storage Account.
```hcl
terraform {
  required_version = "1.9.3"
  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "3.110.0"
    }
    azapi = {
      source = "azure/azapi"
      version = "1.14.0"
    }
  }

  backend "azurerm" {
    subscription_id      = <subscription_id>
    resource_group_name  = <resource_group_name>
    storage_account_name = <storage_account_name>
    container_name       = <container_name >
    key                  = <key>
    use_azuread_auth     = true
  }
}
```
{: file="terraform.tf" }

And following the provider configuration:
```hcl
provider "azurerm" {
  tenant_id           = <tenant_id>
  subscription_id     = <subscription_id>
  storage_use_azuread = true
  
  features {}
}

provider "azapi" {
  tenant_id           = <tenant_id>
  subscription_id     = <subscription_id>
}
```
{: file="provider.tf" }

## Container Registry
In order to create our own container image which will run the agent, we need a Azure Container Registry to store it and make it available for the Container App Job. The terraform code for it is quit simple. It contains also the resource group which will hold all the resources and the user assigned identity with the appropriate role assignements in order for the Container App Job to pull the image and for you to push the image later:
```hcl
resource "azurerm_resource_group" "main" {
  name     = "rg-hosted_agents"
  location = <location>
}

resource "azurerm_user_assigned_identity" "main" {
  name                = "id-hosted_agents"
  location            = azurerm_resource_group.main.location
  resource_group_name = azurerm_resource_group.main.name
}

resource "azurerm_container_registry" "main" {
  name                   = "crhostedagents"
  resource_group_name    = azurerm_resource_group.main.name
  location               = azurerm_resource_group.main.location
  sku                    = "Standard"
  anonymous_pull_enabled = false
}

resource "azurerm_role_assignment" "acr_pull" {
  scope                = azurerm_container_registry.main.id
  role_definition_name = "AcrPull"
  principal_id         = azurerm_user_assigned_identity.main.principal_id
}

resource "azurerm_role_assignment" "user_push" {
  scope                = azurerm_container_registry.main.id
  role_definition_name = "AcrPush"
  principal_id         = [object id of your user or a group you're a member of]
}
```

> Althoug not necesary, with the Premium sku on the Azure Container Registry it would be possible to intgration this service in the vnet as well.
{: .prompt-tip}

## Docker Container Image
With the Container Registry in place we can now create the Agent image using Docker. For the base I used the configuration of this [Microsoft guide about running agents in Docker](https://learn.microsoft.com/en-us/azure/devops/pipelines/agents/docker?view=azure-devops). Create the following `azp-agent.dockerfile`{: .filepath} with one added line for the installation of the Azure CLI as we will use it for authenticaten:
```docker
FROM ubuntu:22.04
ENV TARGETARCH="linux-x64"
# Also can be "linux-arm", "linux-arm64".

RUN apt update
RUN apt upgrade -y
RUN apt install -y curl git jq libicu70

# Install Azure CLI
RUN curl -sL https://aka.ms/InstallAzureCLIDeb | bash

WORKDIR /azp/

COPY ./start.sh ./
RUN chmod +x ./start.sh

# Create agent user and set up home directory
RUN useradd -m -d /home/agent agent
RUN chown -R agent:agent /azp /home/agent

USER agent
# Another option is to run the agent as root.
# ENV AGENT_ALLOW_RUNASROOT="true"

ENTRYPOINT [ "./start.sh" ]
```
{: file="azp-agent.dockerfile" }

Furthermore create the `start.sh`{: .filepath} in the same path as the Dockerfile. Compared with the version of Microsotf it uses a environment variable called `AZP_MI_ID` to pass the ID of the Managed Identity and use it to authenticate to Azure DevOps instead of a personal access token.
```bash
#!/bin/bash
set -e

echo "Starting Azure Pipelines agent..."

if [ -z "${AZP_URL}" ]; then
  echo 1>&2 "error: missing AZP_URL environment variable"
  exit 1
fi

# Check for managed identity ID environment variable and use it for az login
if [ -z "${AZP_MI_ID}" ]; then
  echo "Using managed identity (ID: ${AZP_MI_ID}) for Azure DevOps login..."
  az login --identity --username "${AZP_MI_ID}" --allow-no-subscriptions
else
  echo 1>&2 "error: missing AZP_MI_ID environment variable"
  exit 1
fi

# Generate access token for Azure DevOps
echo "Logging in to Azure DevOps..."
AZP_TOKEN=$(az account get-access-token --resource 499b84ac-1321-427f-aa17-267ca6975798 --query "accessToken" --output tsv)

if [ -z "${AZP_TOKEN_FILE}" ]; then
  if [ -z "${AZP_TOKEN}" ]; then
    echo 1>&2 "error: missing AZP_TOKEN environment variable"
    exit 1
  fi

  AZP_TOKEN_FILE="/azp/.token"
  echo -n "${AZP_TOKEN}" > "${AZP_TOKEN_FILE}"
fi

unset AZP_TOKEN

if [ -n "${AZP_WORK}" ]; then
  mkdir -p "${AZP_WORK}"
fi

cleanup() {
  # If $AZP_PLACEHOLDER is set, skip cleanup
  if [ -n "$AZP_PLACEHOLDER" ]; then
    echo 'Running in placeholder mode, skipping cleanup'
    return
  fi
  if [ -e config.sh ]; then
    print_header "Cleanup. Removing Azure Pipelines agent..."

    # If the agent has some running jobs, the configuration removal process will fail.
    # So, give it some time to finish the job.
    while true; do
      ./config.sh remove --unattended --auth PAT --token $(cat "$AZP_TOKEN_FILE") && break

      echo "Retrying in 30 seconds..."
      sleep 30
    done
  fi
}

print_header() {
  lightcyan="\033[1;36m"
  nocolor="\033[0m"
  echo -e "\n${lightcyan}$1${nocolor}\n"
}

# Let the agent ignore the token env variables
export VSO_AGENT_IGNORE="AZP_TOKEN,AZP_TOKEN_FILE"

print_header "1. Determining matching Azure Pipelines agent..."

AZP_AGENT_PACKAGES=$(curl -LsS \
    -u user:$(cat "${AZP_TOKEN_FILE}") \
    -H "Accept:application/json;" \
    "${AZP_URL}/_apis/distributedtask/packages/agent?platform=${TARGETARCH}&top=1")

AZP_AGENT_PACKAGE_LATEST_URL=$(echo "${AZP_AGENT_PACKAGES}" | jq -r ".value[0].downloadUrl")

if [ -z "${AZP_AGENT_PACKAGE_LATEST_URL}" -o "${AZP_AGENT_PACKAGE_LATEST_URL}" == "null" ]; then
  echo 1>&2 "error: could not determine a matching Azure Pipelines agent"
  echo 1>&2 "check that account "${AZP_URL}" is correct and the token is valid for that account"
  exit 1
fi

print_header "2. Downloading and extracting Azure Pipelines agent..."

curl -LsS "${AZP_AGENT_PACKAGE_LATEST_URL}" | tar -xz & wait $!

source ./env.sh

trap "cleanup; exit 0" EXIT
trap "cleanup; exit 130" INT
trap "cleanup; exit 143" TERM

print_header "3. Configuring Azure Pipelines agent..."

./config.sh --unattended \
  --agent "${AZP_AGENT_NAME:-$(hostname)}" \
  --url "${AZP_URL}" \
  --auth "PAT" \
  --token $(cat "${AZP_TOKEN_FILE}") \
  --pool "${AZP_POOL:-Default}" \
  --work "${AZP_WORK:-_work}" \
  --replace \
  --acceptTeeEula & wait $!

print_header "4. Running Azure Pipelines agent..."

chmod +x ./run.sh

# If $AZP_PLACEHOLDER is set, skipping running the agent
if [ -n "$AZP_PLACEHOLDER" ]; then
  echo 'Running in placeholder mode, skipping running the agent'
else
# To be aware of TERM and INT signals call ./run.sh
# Running it with the --once flag at the end will shut down the agent after the build is executed
  ./run.sh --once & wait $!
fi
```
{: file="start.sh" }

Now startup Docker Desktop and within that same directory run:
```bash
docker build --tag "crhostedagents/azp-agent:0.0.1" --file "./azp-agent.dockerfile" .
```
Then authenticate and push the image to the created Container Registry:
```bash
az login
az acr login --name crhostedagents

docker push "crhostedagents/azp-agent:0.0.1"
```

## Container App Environment