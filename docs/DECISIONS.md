# DECISIONS.md

This document records the major architectural decisions made for Project A, including the reasoning behind each choice, alternatives considered, and trade-offs accepted.

---

## 1. Region Selection - us-east-1 Primary / eu-west-1 DR

### Decision: Use us-east-1 as the primary production and development region, with eu-west-1 serving as the disaster recovery region.

**Chose:** 
- `us-east-1` as the primary region, hosting both development and production workloads.
- `eu-west-1` as the secondary region, hosting production resources for disaster recovery.
- Both regions use two Availability Zones to provide availability within their respective regions.

**Why:**
- Establishes a multi-region architecture capable of recovering from a regional outage.
- Separates the primary production environment from its disaster recovery environment.
- Allows development and production workloads to operate in the primary region without requiring a second development environment.
- Provides a practical way to demonstrate regional resilience and recovery as part of the project's acceptance criteria.

**Alternatives considered:**
- Single-region deployment with multiple Availability Zones.
- Active-active deployment, where both regions handle live production traffic.
- Cold standby, where the DR infrastructure is created or activated only after a failure.

**Trade-off:**
- Operating infrastructure in two regions increases complexity and cost compared with a single-region deployment.
- Cross-region data synchronization and recovery procedures require additional configuration and testing.
- The warm standby approach reduces DR compute costs but does not provide the same immediate capacity as a fully active production region.

---

## 2. Account Topology - Multi-Account Architecture via AWS Organizations

### Decision: Use four separate AWS accounts managed through AWS Organizations.

**Chose:**
- Management account - organization governance and centralized billing.
- Security-Logging account - centralized security monitoring, log aggregation, and immutable log storage.
- Development account - isolated development and testing workloads.
- Production account - live workloads in the primary region and production DR resources in the secondary region.

**Why:**
- Separates administrative responsibilities from application workloads.
- Limits the potential impact of a compromised development or production environment.
- Provides a dedicated location for centralized security logs, reducing the ability of compromised workload accounts to alter historical audit records.
- AWS Organizations allows organization-wide governance through Service Control Policies (SCPs).
- Supports the project's requirement for strong development/production isolation.

**Alternatives considered:**
- A single AWS account containing all environments.
- Two accounts separating development and production, with security logging hosted in one of them.
- Three accounts separating management, workloads, and security logging.

**Trade-off:**
- Managing multiple accounts introduces additional IAM, networking, logging, and Terraform configuration complexity.
- Cross-account permissions must be configured carefully to avoid unintended access.
- Some AWS services and resources may incur additional costs across accounts.
- The management account becomes a highly privileged component that requires particularly strict access controls.

---

## 3. Terraform State Strategy - S3 Remote State with DynamoDB Locking

### Decision: Store Terraform state remotely in Amazon S3 and use DynamoDB for state locking, with separate state isolation for each environment.

**Chose:**
- Amazon S3 as the remote Terraform state backend.
- DynamoDB for state locking to prevent concurrent Terraform operations.
- Separate state files for the different environments and infrastructure deployments.
- Restrictive IAM permissions and encryption to protect access to the state backend.

**Why:**
- Terraform state contains sensitive infrastructure information, including resource identifiers, configuration details, and potentially sensitive values.
- Remote state provides a centralized location for authorized deployment workflows to access infrastructure state.
- State locking helps prevent simultaneous Terraform operations from corrupting or conflicting with the state.
- Environment-specific state isolation reduces the risk of an operation targeting the wrong environment.
- This approach supports automated deployments through the existing CI/CD pipeline.

**Alternatives considered:**
- Local Terraform state stored on an engineer's workstation.
- A remote backend without state locking.
- Terraform Cloud as the remote state management solution.

**Trade-off:**
- The S3 backend and DynamoDB locking introduce additional infrastructure that must itself be secured and maintained.
- Remote state access requires carefully configured IAM permissions and backend configuration.
- Separate state files increase the number of Terraform configurations and deployment operations that must be managed.
- State recovery and version management must be considered if the backend is accidentally modified or deleted.

---

## 4. Environment Isolation - Separate AWS Accounts for Development and Production

### Decision: Isolate development and production workloads using separate AWS accounts rather than relying solely on VPC separation within one account.

**Chose:**
- A dedicated Development account for development workloads.
- A dedicated Production account for live workloads.
- Separate VPCs with non-overlapping CIDR ranges for the different environments.
- Separate IAM deployment roles and permissions for development and production.
- No direct development-to-production administrative access.

**Why:**
- AWS account separation creates a stronger security boundary than relying exclusively on network segmentation.
- A compromised development environment should not automatically provide access to production resources.
- Separate IAM roles allow development and production deployment permissions to be controlled independently.
- Independent account boundaries reduce the blast radius of mistakes, compromised credentials, and unauthorized changes.
- This directly supports the project's requirement for development/production isolation.
- This decision exists independently of the broader account topology (Decision #2). Even if dev and prod theoretically shared the same AWS account, they would still need to be isolated because of threat vectors specific to this boundary — notably the SSRF lateral-movement path identified in THREAT_MODEL.md (Public Subnet → Private Subnet, Threat 1), where a compromised public-facing resource could pivot into adjacent private infrastructure. Account-level isolation closes that path entirely, rather than relying on network controls (security groups, NACLs) alone to prevent it.

**Alternatives considered:**
- Separate development and production VPCs within a single AWS account.
- Separate subnets and security groups within one shared VPC.
- Using separate accounts only for production while hosting development and testing together.

**Trade-off:**
- Separate accounts increase administrative overhead and require cross-account role configuration.
- Resources and services that need to communicate across environments require carefully controlled access.
- Account-level monitoring, logging, and governance must be configured consistently.
- Some shared infrastructure and operational processes become more complicated to maintain.

---

## 5. IAM Federation - GitHub Actions OIDC with Short-Lived AWS Credentials

### Decision: Use OpenID Connect (OIDC) federation for GitHub Actions to assume environment-specific AWS IAM deployment roles instead of storing long-lived AWS access keys.

**Chose:**
- Configure an OIDC identity provider for GitHub Actions.
- Create separate IAM deployment roles for development and production.
- Scope trust policies to the intended repository and deployment context.
- Restrict each role to the permissions required for its deployment tasks.
- Require production environment protection rules and approval before production deployment.

**Why:**
- Long-lived AWS access keys stored in CI/CD secrets create an unnecessary credential exposure risk.
- OIDC allows GitHub Actions to obtain temporary AWS credentials without storing permanent access keys.
- Separate deployment roles prevent development workflows from automatically inheriting production permissions.
- Restricting OIDC trust policies reduces the possibility of unauthorized workflows assuming privileged AWS roles.
- This supports the project's automated deployment requirement while maintaining least-privilege access.

**Alternatives considered:**
- Storing long-lived AWS access keys as GitHub repository or environment secrets.
- Using a manually managed IAM user for deployment.
- Using a different federated identity provider for CI/CD authentication.

**Trade-off:**
- OIDC requires additional configuration between GitHub and AWS, including identity provider setup and precise IAM trust policies.
- Incorrect trust-policy conditions can either prevent legitimate deployments or allow unintended access.
- Production deployments depend on correctly configured GitHub environment protection and approval rules.
- A compromised GitHub repository or workflow can still present a risk even when short-lived credentials are used.

---

## 6. Disaster Recovery Approach - Warm Standby

### Decision: Use a warm-standby disaster recovery model in eu-west-1 rather than active-active or cold standby.

**Chose:**
- `us-east-1` as the active production region handling normal user traffic.
- `eu-west-1` as the warm standby region, with its production network and supporting infrastructure provisioned.
- The DR application tier maintained at reduced capacity to minimize ongoing costs.
- The DR data tier kept synchronized with the primary production environment.
- Amazon Route 53 used to redirect traffic during a regional failover.
- Database promotion performed as part of the recovery process when the DR database becomes the new primary writable database.

**Why:**
- Warm standby provides a balance between recovery readiness and infrastructure cost.
- Keeping the network and supporting infrastructure provisioned reduces the amount of work required during a regional outage.
- Maintaining synchronized data reduces the amount of data that could potentially be lost during failover.
- Reduced application capacity keeps the DR environment less expensive than a fully active production region.
- The approach is appropriate for demonstrating regional recovery while respecting the project's cost-sensitive startup scenario.

**Alternatives considered:**
- Active-active — both regions continuously serve production traffic.
- Cold standby — infrastructure is provisioned or started only after a disaster.
- Backup and restore — recover the environment using stored backups after a regional failure.

**Trade-off:**
- The DR region incurs ongoing costs even when it is not handling production traffic.
- Reduced application capacity means additional resources must be activated during failover.
- Recovery requires coordinated DNS changes, application scaling, and database promotion.
- Failover is not instantaneous and must be tested to establish realistic Recovery Time Objective (RTO) and Recovery Point Objective (RPO) values.
- The architecture depends on reliable cross-region data synchronization and a documented recovery procedure.