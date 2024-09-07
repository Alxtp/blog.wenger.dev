---
title: "Simple and Secure: Deploy self-hosted Azure DevOps Agents without PAT"
date: 2024-08-29 21:00:00 +0200
categories: [Azure]
tags: [azure, automation, terraform, iac, "azure devops", container, docker]
img_path: /azure/
---

In the world of Azure DevOps, self-hosted agents provide greater control and customization for your CI/CD pipelines. However, traditional deployment methods often rely on personal access tokens (PATs), such as [Microsoft's official documentation on deploying self-hosted agents with container app jobs](https://learn.microsoft.com/en-us/azure/container-apps/tutorial-ci-cd-runners-jobs?tabs=bash&pivots=container-apps-jobs-self-hosted-ci-cd-azure-pipelines) also suggests, which can introduce security risks and are tied to a user account. 

This post explores a modern, secure approach to deploying self-hosted Azure DevOps agents using Azure Container Apps jobs and managed identity, and also using Terraform to automatically create the infrastructure. To also secure network traffic and integrate with other Azure services, the entire setup will be fully vnet integrated. This will require a firewall on a hub network already in place. In this guide I'll use an Azure firewall.

## Prerequisites
- Azure Account
- Azure CLI
- Terraform
- Docker Desktop
- Azure DevOps organization

## Architecture
The final architecture looks like this:

![Architecture Diagramm](architecture.png)
_Architecture Diagramm_
At the core is a **Container Apps Environment** with an **Internal Workload Profile**, which includes:

- **Virtual Network and Subnet** - private network on which the container will run
- **Private Endpoint and Private DNS Zone** - required by the Azure Monitor Private Link Scope
- **Container App Job** - will start up a docker container if there are jobs in the Azure DevOps pool

The Containers connect to an Azure Firewall for managing outgoing traffic.


Furthermore the architecture incorporates:

- **Container Registry** - for storing and managing container images
- **User Assigned Managed Identity** - for access to the Container Registry and to the Agent Pool in Azure DevOps
- **Azure Monitor Private Link Scope** - for private integrration of the Log Analytics Workspace
- **Log Analytics Workspace** - in order to monitor and troublshoot container runs

## Setup Terraform
To deploy the Azure resources, I will use Terraform, which simplifies the creation and management of all resources. See the required providers. The state will be stored in an Azure Storage Account.
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
    container_name       = <container_name>
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
To create our own container image that runs the agent, we need an Azure container registry to store it and make it available to the container app job. The Terraform code for this is quite simple. It also contains the resource group, which will hold all the resources, and the user-assigned identity with the appropriate role assignments so that the Container App Job can pull the image and you can push the image later:

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

> Although not required, with the Premium SKU on the Azure Container Registry, it would be possible to integrate this service into the VNET as well.
{: .prompt-tip}

## Azure DevOps Agent Pool
The next step is to create an agent pool. Since this task is straightforward, you can follow the step-by-step instructions in the Microsoft documentation here: [https://learn.microsoft.com/en-us/azure/devops/pipelines/agents/pools-queues?view=azure-devops&tabs=yaml%2Cbrowser](https://learn.microsoft.com/en-us/azure/devops/pipelines/agents/pools-queues?view=azure-devops&tabs=yaml%2Cbrowser).

Just make sure that you add the previously created Azure User Assigned Managed Identity to your organization and assign the `Administrator` role to it in the Agent Pool (Organization-level), as it is required for registering and unregistering agents.

## Docker Container Image
With the Azure Container Registry in place, we can now create the agent image using Docker. As a base, I used the configuration from this [Microsoft guide to running agents in Docker](https://learn.microsoft.com/en-us/azure/devops/pipelines/agents/docker?view=azure-devops). Create the following `azp-agent.dockerfile`{:.filepath} with an additional line for the installation of the Azure CLI as we will use it for authentication:
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

Furthermore create the `start.sh`{: .filepath} in the same path as the Dockerfile. Compared to the Microsotf version, it uses an environment variable called `AZP_MI_ID` to pass the managed identity ID and use it to authenticate to Azure DevOps instead of a personal access token.
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

Then authenticate and push the image to the created container registry:
```bash
az login
az acr login --name crhostedagents

docker push "crhostedagents/azp-agent:0.0.1"
```

## Network
The networking part is pretty big, because we are fully integrating the services into the Azure Virtual Network, with all communication going through the Azure backbone. It is also integrated into a standard hub-and-spoke topology network with an Azure firewall in place. This also requires peering to the hub and a route table. For logging, you will need a private link scope, a private endpoint, and some private DNS zones. See all the required network resources in this Terraform code:
```hcl
locals {
  private_dns_zones_names = toset([
    "privatelink.agentsvc.azure-automation.net",
    "privatelink.blob.core.windows.net",
    "privatelink.monitor.azure.com",
    "privatelink.ods.opinsights.azure.com",
    "privatelink.oms.opinsights.azure.com"
  ])
}

resource "azurerm_virtual_network" "main" {
  name                = "vnet-hosted_agents"
  resource_group_name = azurerm_resource_group.main.name
  location            = azurerm_resource_group.main.location
  address_space       = ["172.20.0.0/20"]
}

resource "azurerm_subnet" "container_app_environment" {
  name                 = "subnet-hosted_agents"
  resource_group_name  = azurerm_resource_group.main.name
  virtual_network_name = azurerm_virtual_network.main.name
  address_prefixes     = [cidrsubnet(azurerm_virtual_network.main.address_space[0], 7, 0)]

  delegation {
    name = "containerAppEnvironment"

    service_delegation {
      name    = "Microsoft.App/environments"
      actions = ["Microsoft.Network/virtualNetworks/subnets/join/action"]
    }
  }
}

resource "azurerm_subnet" "shared" {
  name                 = "subnet-shared"
  resource_group_name  = azurerm_resource_group.main.name
  virtual_network_name = azurerm_virtual_network.main.name
  address_prefixes     = [cidrsubnet(azurerm_virtual_network.main.address_space[0], 7, 1)]
}

data "azurerm_virtual_network" "hub" {
  name                = "vnet-hub" # Replace with the name of the hub vnet
  resource_group_name = "rg-hub" # Replace with the name of the hub resource group
}

resource "azurerm_virtual_network_peering" "peer_spoke_to_hub" {
  name                         = "peer-spoke-to-hub"
  resource_group_name          = azurerm_resource_group.main.name
  virtual_network_name         = azurerm_virtual_network.main.name
  remote_virtual_network_id    = data.azurerm_virtual_network.hub.name
  allow_virtual_network_access = true
  allow_forwarded_traffic      = true
  allow_gateway_transit        = false
  use_remote_gateways          = false
}

resource "azurerm_virtual_network_peering" "peer_hub_to_spoke" {
  name                         = "peer-hub-to-spoke"
  resource_group_name          = azurerm_resource_group.main.name
  virtual_network_name         = data.azurerm_virtual_network.hub.name
  remote_virtual_network_id    = azurerm_virtual_network.main.name
  allow_virtual_network_access = true
  allow_forwarded_traffic      = true
  allow_gateway_transit        = false
  use_remote_gateways          = false
}

resource "azurerm_route_table" "main" {
  name                = "rt-hosted_agents"
  resource_group_name = azurerm_resource_group.main.name
  location            = azurerm_resource_group.main.location
}

resource "azurerm_route" "default" {
  name                   = "azure-firewall-route"
  resource_group_name    = azurerm_resource_group.main.name
  route_table_name       = azurerm_route_table.main.name
  address_prefix         = "0.0.0.0/0"
  next_hop_type          = "VirtualAppliance"
  next_hop_in_ip_address = "172.18.2.4" # replace with the IP of the Firewall
}

resource "azurerm_subnet_route_table_association" "container_app_environment" {
  subnet_id      = azurerm_subnet.container_app_environment.id
  route_table_id = azurerm_route_table.main.id
}

resource "azurerm_monitor_private_link_scope" "main" {
  name                = "ampls-hosted_agents"
  resource_group_name = azurerm_resource_group.main.name

  ingestion_access_mode = "PrivateOnly"
  query_access_mode     = "Open"
}

resource "azurerm_monitor_private_link_scoped_service" "main" {
  name                = "log-analytics"
  resource_group_name = azurerm_resource_group.main.name
  scope_name          = azurerm_monitor_private_link_scope.main.name
  linked_resource_id  = azurerm_log_analytics_workspace.main.id
}

resource "azurerm_private_dns_zone" "azure_monitor" {
  for_each = local.private_dns_zones_names

  name                = each.value
  resource_group_name = azurerm_resource_group.main.name
}

resource "azurerm_private_endpoint" "azure_monitor" {
  name                          = "pep-hosted_agents"
  location                      = azurerm_resource_group.main.location
  resource_group_name           = azurerm_resource_group.main.name
  subnet_id                     = azurerm_subnet.shared.id
  custom_network_interface_name = "nic-hosted_agents"

  private_service_connection {
    name                           = "azure_monitor"
    private_connection_resource_id = azurerm_monitor_private_link_scope.main.id
    subresource_names              = ["azuremonitor"]
    is_manual_connection           = false
  }

  private_dns_zone_group {
    name                 = "dnszonegroups"
    private_dns_zone_ids = [for dns_zone in azurerm_private_dns_zone.azure_monitor : dns_zone.id]
  }
}

resource "azurerm_private_dns_zone_virtual_network_link" "azure_monitor" {
  for_each = azurerm_private_dns_zone.azure_monitor

  name                  = "azure_monitor"
  resource_group_name   = azurerm_resource_group.main.name
  private_dns_zone_name = each.value.name
  virtual_network_id    = azurerm_virtual_network.main.id
}
```

### Azure Firewall Rules
Now that all the network components are in place, we need to create some firewall rules on the Azuire firewall to allow the traffic that will be required by the Container App environment and the Container App job to work. The required rules are [documented by Microsoft](https://learn.microsoft.com/en-us/azure/container-apps/networking?tabs=workload-profiles-env%2Cazure-cli#configuring-udr-with-azure-firewall), but as I inspected the firewall logs I had to add some more rules. In the end I came up with the following rules:

- **Application** (80,443): `crhostedagents.azurecr.io`, `*.blob.core.windows.net`, `login.microsoft.com`, `mcr.microsoft.com`, `*.data.mcr.microsoft.com`, `*.identity.azure.net`, `login.microsoftonline.com`, `*.login.microsoftonline.com`, `*.login.microsoft.com`, `raw.githubusercontent.com`, `dc.services.visualstudio.com`, `management.azure.com`, `vstsagentpackage.azureedge.net`, `crl.microsoft.com`, `*.azureedge.net`, `packages.microsoft.com`
- **Network** (80,443) Service Tags: `MicrosoftContainerRegistry`, `AzureFrontDoor.FirstParty`, `AzureContainerRegistry`, `AzureActiveDirectory`


## Container App Environment
Let's move on to the core of the infrastructure: The Container App Environment with the Log Analytics Workspace. There are some special considerations here because the Container App job, especially the Managed Identity integration, is still in heavy development.

> Note that it is necessary to create a Placholder agent and run it once. After this run, the agent will show up in the Azure DevOps pool and this resource can be deleted. This is documented here: [https://learn.microsoft.com/en-us/azure/container-apps/tutorial-ci-cd-runners-jobs?tabs=bash&pivots=container-apps-jobs-self-hosted-ci-cd-azure-pipelines#create-a-placeholder-self-hosted-agent](https://learn.microsoft.com/en-us/azure/container-apps/tutorial-ci-cd-runners-jobs?tabs=bash&pivots=container-apps-jobs-self-hosted-ci-cd-azure-pipelines#create-a-placeholder-self-hosted-agent)
{: .prompt-info}

> I had to use the azapi provider to create the "real" container app job because the scale rule in the resource `azurerm_container_app_job` doesn't support managed identity yet: [https://github.com/hashicorp/terraform-provider-azurerm/issues/26570](https://github.com/hashicorp/terraform-provider-azurerm/issues/26570). At this time, this option is only available in the `2024-02-02-preview` API version implemented here: [https://github.com/microsoft/azure-container-apps/issues/592](https://github.com/microsoft/azure-container-apps/issues/592)
{: .prompt-info}

> To use Managed Identities inside Container Apps or Jobs, the environment variable `APPSETTING_WEBSITE_SITE_NAME` with a random value is required: [https://github.com/Azure/azure-cli/issues/22677](https://github.com/Azure/azure-cli/issues/22677)
{: .prompt-info}

```hcl
resource "azurerm_log_analytics_workspace" "main" {
  name                            = "log-hostedagents"
  location                        = azurerm_resource_group.main.location
  resource_group_name             = azurerm_resource_group.main.name
  allow_resource_only_permissions = true
  local_authentication_disabled   = true
  sku                             = "PerGB2018"
}

resource "azurerm_container_app_environment" "main" {
  name                               = "cae-hostedagents"
  location                           = azurerm_resource_group.main.location
  resource_group_name                = azurerm_resource_group.main.name
  infrastructure_resource_group_name = "rg-managed_app_env"
  mutual_tls_enabled                 = true
  infrastructure_subnet_id           = azurerm_subnet.container_app_environment.id
  internal_load_balancer_enabled     = true
  log_analytics_workspace_id         = azurerm_log_analytics_workspace.main.id

  workload_profile {
    name                  = "Consumption"
    workload_profile_type = "Consumption"
    minimum_count         = 0
    maximum_count         = 0
  }
}

# The placeholder agent is required for the agent pool to work. After creaton of the agent in the pool, this resource can be removed.
resource "azurerm_container_app_job" "placeholder" {
  name                         = "placeholder-agent-job"
  location                     = azurerm_resource_group.main.location
  resource_group_name          = azurerm_resource_group.main.name
  container_app_environment_id = azurerm_container_app_environment.main.id
  workload_profile_name        = "Consumption"

  replica_timeout_in_seconds = 300
  replica_retry_limit        = 0

  manual_trigger_config {
    parallelism              = 1
    replica_completion_count = 1
  }

  template {
    container {
      image  = "crhostedagents.azurecr.io/azp-agent:0.0.1"
      name   = "agent"
      cpu    = 0.5
      memory = "1Gi"

      env {
        name        = "AZP_URL"
        secret_name = "devops-org-url"
      }

      env {
        name  = "AZP_POOL"
        value = "container-apps" # Replace with your Agent Pool name
      }

      env {
        name  = "AZP_PLACEHOLDER"
        value = 1
      }

      env {
        name  = "AZP_AGENT_NAME"
        value = "placeholder-agent"
      }

      env {
        name  = "AZP_MI_ID"
        value = azurerm_user_assigned_identity.main.client_id
      }

      env {
        name  = "APPSETTING_WEBSITE_SITE_NAME" # Required for authentication with MSI
        value = "placeholder-agent"
      }
    }
  }

  secret {
    name  = "devops-org-url"
    value = "https://dev.azure.com/<org_name>" # Replace with your Azure DevOps organization url
  }

  identity {
    type         = "UserAssigned"
    identity_ids = [azurerm_user_assigned_identity.main.id]
  }

  registry {
    server   = azurerm_container_registry.main.login_server
    identity = azurerm_user_assigned_identity.main.id
  }
}

resource "azapi_resource" "agent" {
  type = "Microsoft.App/jobs@2024-02-02-preview"
  name = "ca-hostedagents"

  parent_id = azurerm_resource_group.main.id

  body = {

    location = azurerm_resource_group.main.location
    properties = {
      environmentId       = azurerm_container_app_environment.main.id
      workloadProfileName = "Consumption"
      configuration = {
        secrets = [
          {
            name  = "devops-org-url"
            value = "https://dev.azure.com/<org_name>" # Replace with your Azure DevOps organization url
          }
        ]
        triggerType           = "Event"
        replicaTimeout        = 1800
        replicaRetryLimit     = 0
        manualTriggerConfig   = null
        scheduleTriggerConfig = null
        eventTriggerConfig = {
          replicaCompletionCount = 1
          parallelism            = 1
          scale = {
            minExecutions   = 0
            maxExecutions   = 10
            pollingInterval = 30
            rules = [
              {
                name = "azure-pipelines"
                type = "azure-pipelines"
                metadata = {
                  poolName = "container-apps" # Replace with your Agent Pool name
                }
                auth = [
                  {
                    secretRef        = "devops-org-url"
                    triggerParameter = "organizationURL"
                  }
                ]
                identity = azurerm_user_assigned_identity.main.id
              }
            ]
          }
        }
        registries = [
          {
            server            = "crhostedagents.azurecr.io"
            username          = ""
            passwordSecretRef = ""
            identity          = azurerm_user_assigned_identity.main.id
          }
        ]
        identitySettings = []
      }
      template = {
        containers = [
          {
            image     = "crhostedagents.azurecr.io/azp-agent:0.0.1"
            imageType = "ContainerImage"
            name      = "agent"
            env = [
              {
                name      = "AZP_URL"
                secretRef = "devops-org-url"
              },
              {
                name  = "AZP_POOL"
                value = "container-apps" # Replace with your Agent Pool name
              },
              {
                name  = "AZP_MI_ID"
                value = azurerm_user_assigned_identity.main.client_id
              },
              {
                name  = "APPSETTING_WEBSITE_SITE_NAME" # Required for Authentication with MSI
                value = "agent"
              }
            ]
            resources = {
              cpu    = 2
              memory = "4Gi"
            }
            probes = []
          }
        ]
        initContainers = null
        volumes        = []
      }
    }
    identity = {
      type = "UserAssigned"
      userAssignedIdentities = {
        (azurerm_user_assigned_identity.main.id) = {
        }
      }
    }
  }
}
```

## Conclusion

With the infrastructure in place, you're now ready to utilize your newly created self-hosted agents. To get started:

1. Run the placeholder Container App Job (the agent should appear in the Agent Pool as offline)
2. Delete the placeholder Container App Job
3. Assign the created Agent Pool to a pipeline by specifying `pool: <pool_name>` in the YAML configuration
4. Run the pipeline

That's it! Azure DevOps will now place jobs in the designated Agent Pool. The Container App Job checks this pool every 30 seconds for new jobs, spinning up an agent to perform each task and destroying it afterward. You can monitor the executed jobs in the execution history of the Container App Job.

This modern approach to self-hosted Azure DevOps agents gives you a secure and scalable CI/CD environment that also helps you integrate with other Azure services that are VNET-integrated.