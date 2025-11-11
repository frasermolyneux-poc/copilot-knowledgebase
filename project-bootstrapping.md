# Project Bootstrapping

## Meta Terraform (.github)
- The `.github/terraform` solution is the control plane for every proof of concept (PoC) repository. It targets Azure (`azurerm` provider) and GitHub (`integrations/github` provider) and stores its own state in an Azure Storage account via `backend "azurerm"`.
- Environment characteristics live in `projects/*.json`. Each file describes the GitHub repo metadata plus one or more named environments. Environments can opt into managed Terraform state (`configure_terraform_state`), define spoke virtual networks (`vnet_spokes`), and request additional resource groups (`resource_groups`).
- Shared settings (subscription, hub virtual network, DNS zones, meta-state storage) are injected through `tfvars/prd.tfvars`. This allows the same plan to be replayed against short-lived tenants by simply swapping the variable file.

## Azure Foundations Provisioned
### Terraform State
- A shared resource group `rg-alz-workload-terraform-<location>` holds per-environment Storage accounts when `configure_terraform_state` is true. Account names use `random_id.environment` to satisfy global uniqueness and expose a private container named `tfstate`.
- Each Storage account gains a private endpoint in the workload-vending VNet (`var.private_endpoint_vnet_name`) and links to the central blob private DNS zone so backends stay private.
- Service principals receive `Storage Account Key Operator Service Role` and `Storage Blob Data Contributor` on their dedicated backend. If a separate meta-state store is declared (`meta_state_*` variables), they also gain `Storage Blob Data Reader` there to support state fan-out across subscriptions.

### Resource Groups and VNets
- For every environment/location combination, the meta project emits predictable names:
  - Resource groups: `rg-<project>-<environment>-<location>` (from `resource_group_definitions`).
  - Virtual networks: `vnet-<project>-<environment>-<location>` (from `workload_spoke_definitions`).
- Workload spokes automatically peer to the GitHub runner hub VNet (`var.github_vnet_name`) in both directions, creating a hub-and-spoke topology usable by self-hosted runners.
- Resource groups listed explicitly under `resource_groups` reuse the same naming pattern and tags but do not create VNets.

### Private DNS
- Central private DNS zones (e.g. `privatelink.azurewebsites.net`, `privatelink.blob.core.windows.net`) are declared once in `private_dns.tf`.
- Every workload VNet is linked to each managed zone with registration disabled by default, providing private name resolution for common Azure PaaS services without duplicating zones per project.

## Identity and Access Model
- Each environment emits an Entra application and service principal (`spn-github-<project>-<environment>`). OIDC federated identity credentials are bound to the repo/environment pair (`repo:frasermolyneux-poc/<project>:environment:<environment>`), enabling GitHub Actions sign-in without secrets.
- Subscription-level roles applied to every principal:
  - `Contributor`
  - `Key Vault Administrator`
  - `Role Based Access Control Administrator` limited by the ABAC condition in `service_principal_role_assignment.tf` to prevent assignment of overly permissive roles.
- Shared tags propagate across resource groups, VNets, and Storage accounts to keep cost-governance reporting consistent.

## GitHub Integration
- `github_repository.tf` creates or adopts repositories using the metadata from each JSON definition (topics, template source, visibility, branch hygiene).
- `github_repository_environment.tf` provisions GitHub environments per Terraform environment (for example, `poc`). Secrets seeded into each environment include:
  - `AZURE_CLIENT_ID`, `AZURE_SUBSCRIPTION_ID`, `AZURE_TENANT_ID` for OIDC sign-in.
  - When Terraform state is managed by the meta project: `tf_backend_subscription_id`, `tf_backend_resource_group_name`, `tf_backend_storage_account_name`, `tf_backend_container_name`, and `tf_backend_key`.
- Dependabot receives matching secrets for the `poc` environment via `github_dependabot_secrets.tf`, so security updates can reuse the same identity.

## Consuming the Foundation in Workloads
- Downstream projects import the foundation outputs through a `data "terraform_remote_state" "foundation"` block (see `az-functions-secure-config/terraform/remote_state.tf`). Backend connection parameters map directly to the secrets emitted above.
- Each workload computes a key using `format("%s-%s", var.workload_name, var.environment_name)` and combines it with the deployment region to read its slice of the `workload_spokes` output. The payload exposes:
  - The workload resource group(s) keyed by location.
  - The shared VNet properties (name, id, address space).
  - The hub peering identifiers and DNS zone resource group for private link resources.
- Projects then extend the shared VNet (for example, adding subnets) or deploy resources into the pre-created resource group while continuing to use the centrally managed backend and identity.

## Meta-State and Multi-Tenant Support
- When PoCs must read Terraform state hosted in another subscription, set `meta_state_subscription_id`, `meta_state_resource_group_name`, and `meta_state_storage_account_name` in the `.github` `tfvars`. The meta run will grant `Storage Blob Data Reader` on that store to every service principal, eliminating per-project maintenance.
- Because the meta definition is data-driven, onboarding a new tenant only requires updating the variable file and re-running Terraform; the project JSON stays portable across tenants.

## Operational Notes
- `random_id.environment_id` and `time_rotating.thirty_days` exist to support unique naming and optional secret rotation triggers for future enhancements.
- Naming truncation guards (`substr`) keep resource names within Azure limits while preserving the project/environment context.
- To add a project: copy an existing JSON file, adjust the metadata, supply spoke address space(s), decide whether the meta layer should manage the backend, then run `terraform apply` in `.github/terraform`. Downstream repositories consume the new foundation automatically once their workflows reference the emitted secrets.
