# Microsoft Fabric Platform Startup Plan — Production-Ready (CI/CD + Service Principals)
This README documents a complete, hand-driven startup plan for standing up a Microsoft Fabric platform in Azure. It assumes you have an Azure account and a Fabric-enabled Entra ID tenant but do not yet have an 
Azure subscription or Fabric capacity. This guide includes operational controls (cost, monitoring, backups) and security best practices. 
Can be used as a checklist.

## Scope and Assumptions

Scope: everything is done manually through the Azure and Fabric portals.
Roles required across the process:
•	Global Administrator or Billing Administrator — to create the subscription.
•	Entra ID Administrator — to create groups and app registrations.
•	Fabric Administrator — to change tenant settings and allow SP access.
•	Capacity / Subscription Owner or Contributor — to create the Fabric capacity.

## Pre-flight decisions

•	Environment naming convention: platform-{env}-{role} (e.g., sg-fabric-platform-dev-admins). Use consistent suffixes for dev/test/prod.
•	Networking posture: public endpoints only, or require private connectivity (managed private endpoints, gateway, or VNet integration)?
•	Data residency/region: pick the region closest to users/data and use the same region for resource group and capacity.
•	Cost guardrails: initial SKU (F2), monthly budget threshold, auto-pause policy for non-prod.
•	Backup/versioning stance: Git integration mandatory for Fabric items; retention and Delta time travel policy for data.
•	CI/CD model: enforce per-stage service principals and federated credentials where possible.

## High-level step sequence

Follow these steps in order. Each step includes actions, recommended decisions, and who should perform them depending on role ownership.
1.	Create an Azure subscription (Billing or Global Admin). If your organization owns subscriptions, request Contributor/Owner access on the target subscription instead of making a new one.
2.	Create a resource group for Fabric capacity and tag it for cost tracking (Environment, Project, CostCenter).
3.	Create Entra ID security groups (tenant-level where appropriate). Create them empty.
4.	Assign the Fabric Administrator Entra role to 1–3 named individuals or to the sg-fabric-capacity-admins group (Entra Admin / Fabric Admin). Avoid broad assignments.
5.	Create the Fabric capacity (Subscription Owner/Contributor). Choose a region, SKU (start with F2), and administrators (add sg-fabric-capacity-admins). Immediately create an Azure Budget on the resource group
    (Cost Management → Budgets) and set alerts at 50/80/100%.
6.	Configure Fabric tenant settings (Fabric Administrator). Enable service principals to use Fabric APIs and scope that setting to sg-fabric-service-principals. Confirm "Users can create Fabric items" as needed.
    Confirm capacity is listed.
7.	Create the Fabric workspace(s) (workspace Admin). Choose license mode → Fabric capacity and select the capacity created. For multiple environments, create platform-dev, platform-test, platform-prod as separate
    workspaces.
8.	Configure workspace settings (workspace Admin): OneLake access stance, sensitivity labels policy, default Spark pool settings, and Git integration policy. Document decisions.
9.	Assign Entra groups to workspace roles (workspace Admin): map sg-fabric-platform-{env}-admins → Admin, -members → Member, -contributors → Contributor, -viewers → Viewer. Add sg-fabric-service-principals as
    Contributor/Member as appropriate.
10.	Add people (Entra Admin / Group Owners) to the groups (populate membership). Keep groups empty until this step to preserve separation between role plumbing and membership decisions.
11.	Create first Fabric items (workspace Admin/Contributors): create a Lakehouse for landing data, then notebooks, data pipelines, and a semantic model/report. Use Git integration from step 9 for version control.
12.	Implement CI/CD and service principals. See the detailed section below.

## Entra ID security group mapping (table)

| Group Name | Scope | Fabric Role Mapped | Who Goes in It |
|------------|--------|-------------------|----------------|
| sg-fabric-capacity-admins | Tenant / Capacity | Capacity Administrator (Azure resource) + optional Fabric Administrator | Platform/Infrastructure owners (2–3 people max) |
| sg-fabric-platform-dev-admins | Per Workspace | Admin | Platform engineering leads for the development workspace |
| sg-fabric-platform-dev-members | Per Workspace | Member | Senior engineers for the development workspace |
| sg-fabric-platform-dev-contributors | Per Workspace | Contributor | Day-to-day engineers for the development workspace |
| sg-fabric-platform-dev-viewers | Per Workspace | Viewer | Stakeholders and auditors |
| sg-fabric-service-principals | Per Tenant or Per Workspace | Contributor or Member (for automation) | Service principals used by CI/CD pipelines and automation (no human users) |

## Service principals and CI/CD

CI/CD and service principals are required for this production-ready plan. The following prescribes a secure, least-privilege setup using federated credentials when possible.
1.	Register the app in Entra ID (Entra Admin): App registrations → + New registration. Name format: sp-fabric-deploy-{env} (e.g., sp-fabric-deploy-dev). Supported account types: Accounts in this organizational
    directory only. Note the Application (client) ID and Directory (tenant) ID.
2.	Add a federated credential (preferred) for GitHub Actions: App registration → Certificates & secrets → Federated credentials → + Add credential. Choose the scenario (GitHub Actions)
    and scope to the repo and branch/environment for that deployment stage. For prod, scope to a protected environment/branch only.
3.	If federated credentials are not available for a runner, create a client secret with a short expiry (90 days) and store it in a secret store (Azure Key Vault, GitHub secrets). Document the rotation policy and set
    calendar reminders.
4.	Add the app to sg-fabric-service-principals as a member.
5.	Enable service principal access in the Fabric Admin portal (Fabric Administrator): Tenant settings → Developer settings → Service principals can use Fabric APIs → ON and scope to sg-fabric-service-principals.
6.	Grant workspace role to the group (Workspace Admin): workspace → Manage access → add sg-fabric-service-principals → Contributor (or Member) — give the least privilege necessary.
7.	If the automation needs access to Azure resources (e.g., storage), give the SP the specific Azure RBAC role required on that resource (Storage Blob Data Contributor). Use Enterprise applications to find the
    service principal object and assign RBAC scoped to the resource (not subscription-wide).
8.	Adopt per-stage SPs: at minimum have two SPs: one for dev and one for prod. Federate each only from the appropriate CI environment/branch.

## CI/CD recommendations and workflow

•	Repository layout: a single repo with folders per workspace and manifests for Lakehouse schemas, notebooks, pipelines, semantic models, and deployment manifests.
•	Branch strategy: short-lived feature branches, protected main branch for prod, and a dev branch for continuous integration. Use protected environments for any production-sensitive deploys.
•	CI pipeline: lint, static checks, unit tests (where applicable), validate deployment manifests against a schema, run a dry-run or plan step that does not modify Fabric.
•	CD pipeline: triggered on push to dev/test/main or via pull request approvals. Uses the corresponding SP via federated credential to authenticate and apply changes to the Fabric workspace via the Fabric REST API 
    or Fabric's publish/deploy endpoints.
•	Approval gates: require at least one approver from the platform team for prod deploys and use environment-based approvals where your CI/CD system supports them.



