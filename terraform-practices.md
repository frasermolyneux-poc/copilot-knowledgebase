# Terraform Practices

## Shared Patterns
- Terraform state lives in Azure Storage via the `azurerm` backend; every environment can request a dedicated Storage account and private endpoint using `configure_terraform_state` in the project JSON.
- Providers are centralised in `.github/terraform/providers.tf`, pinning versions and enabling AzureAD auth for storage:
  ```hcl
  terraform {
    required_providers {
      azurerm = {
        source  = "hashicorp/azurerm"
        version = "~> 4.15"
      }
      github = {
        source  = "integrations/github"
        version = "~> 6.5.0"
      }
    }
    backend "azurerm" {}
  }

  provider "azurerm" {
    subscription_id = var.subscription_id
    features {
      resource_group {
        prevent_deletion_if_contains_resources = false
      }
    }
    storage_use_azuread = true
  }
  ```
- Environment data is driven from `projects/*.json`, keeping project definitions declarative and portable; Terraform flattens these into locals for iteration.
- Naming stays consistent through `format("rg-%s-%s-%s", project, environment, location)` and `substr` to respect Azure limits.
- Tags merge global metadata with project-specific context for cost reporting and governance.

## Identity & Access
- Every environment generates a service principal (`azuread_application` + `azuread_service_principal`) and attaches OIDC credentials bound to the GitHub repository environment:
  ```hcl
  resource "azuread_application_federated_identity_credential" "oidc" {
    for_each      = local.project_environments
    application_id = azuread_application.principal[each.key].id
    display_name   = format("github-%s-%s", lower(each.value.project), lower(each.value.environment))
    issuer         = "https://token.actions.githubusercontent.com"
    subject        = format("repo:frasermolyneux-poc/%s:environment:%s", lower(each.value.project), each.value.environment)
    audiences      = ["api://AzureADTokenExchange"]
  }
  ```
- Role assignments deliver least-privilege access needed for PoCs: subscription Contributor, Key Vault Administrator, and a conditioned RBAC Administrator to shield privileged role grants.

## Networking & DNS
- Workload VNets are created per project/environment/location and automatically peered to the central GitHub runner VNet to keep workflows private.
- Private DNS zones are declared once and linked to every workload VNet, avoiding duplicate zone management while enabling private endpoints across services.

## Reusing Foundations in Workloads
- Downstream repositories read the shared state and hydrate local data structures before extending the environment:
  ```hcl
  data "terraform_remote_state" "foundation" {
    backend = "azurerm"
    config = {
      resource_group_name  = var.foundation_state_resource_group_name
      storage_account_name = var.foundation_state_storage_account_name
      container_name       = var.foundation_state_container_name
      key                  = var.foundation_state_key
      use_azuread_auth     = true
    }
  }

  locals {
    workload_state_key    = format("%s-%s", var.workload_name, var.environment_name)
    workload_location_key = format("%s-%s", local.workload_state_key, var.location)
    workload_spoke        = try(data.terraform_remote_state.foundation.outputs.workload_spokes[local.workload_location_key], null)
  }
  ```
- Subnets, private endpoints, and application resources are then layered onto the shared VNet and resource groups supplied by the meta project.

## GitHub Integration
- GitHub environments mirror Terraform environments. Secrets for Azure OIDC sign-in and backend configuration are injected centrally to keep workflow definitions minimal and secret-free.
- Dependabot inherits the same secrets in `poc` environments so automated dependency updates can plan and apply without additional configuration.
