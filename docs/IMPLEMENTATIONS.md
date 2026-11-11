# IMPLEMENTATION_PLAN.md

## Project A — Secure Multi-Region AWS Platform

This document defines the implementation sequence for Project A. Each milestone contains specific tasks, validation procedures, and completion criteria.

The infrastructure will be built incrementally, with each milestone validated before moving to the next.

---
### The issue: Milestone 2 has you deploying Terraform into Dev, Prod, and Security-Logging accounts. Milestone 3 is where you build the OIDC/IAM roles that let Terraform (via CI/CD) assume cross-account access. But Milestone 2 happens before Milestone 3 — so how does Terraform get into those other accounts in Milestone 2, if the permission system to do that isn’t built until Milestone 3?
0. Configure AWS provider `assume_role` blocks in Terraform, using 
   OrganizationAccountAccessRole, to allow the Management-account-authenticated 
   operator (you) to deploy into Dev/Prod/Security-Logging accounts manually, 
   before CI/CD-based OIDC roles exist in Milestone 3.

## Milestone 1: Repository and Terraform Foundation

**Goal:** Establish a structured Terraform repository with a secured remote state backend and automated CI validation.

**Tasks:**

1. Initialize the Git repository and create the folder structure:
   - `/modules`
   - `/environments/dev`
   - `/environments/prod`
   - `/environments/dr`
   - `/bootstrap`
   - `/docs`
2. Add `.terraform-version` and define Terraform and AWS provider version constraints in `versions.tf`.
3. Configure Terraform formatting and validation requirements.
4. Create the initial S3 bucket and DynamoDB table configuration for remote state in the separate bootstrap configuration.
5. Apply restrictive IAM and S3 bucket policies to control access to the remote state backend.
6. Enable S3 versioning and encryption on the state bucket.
7. Configure separate Terraform state configurations for the development, production, and DR environments.
8. Test state locking by attempting concurrent Terraform operations.
9. Create a GitHub Actions workflow that runs `terraform fmt -check` and `terraform validate` on proposed changes.

**Validation:**

- Run `terraform fmt -check` and confirm that the repository passes formatting validation.
- Run `terraform validate` and confirm that the Terraform configurations are valid.
- Attempt concurrent Terraform operations and confirm that state locking prevents conflicting operations.
- Test unauthorized access to the state bucket and confirm that access is denied.
- Create a test pull request and confirm that GitHub Actions executes the validation workflow.

**Done when:**

- The Terraform repository has the required folder structure and version constraints.
- The remote state backend exists, is encrypted, versioned, and access-controlled.
- State locking works correctly.
- CI automatically blocks unformatted or invalid Terraform configurations.

---

## Milestone 2: Network Infrastructure

**Goal:** Build isolated network infrastructure across the development, production, and DR environments, with controlled inbound and outbound traffic.

**Tasks:**

1. Create the initial VPC Terraform module.
2. Define non-overlapping CIDR ranges for the three VPC deployments:
   - Development: `10.0.0.0/16`
   - Production primary: `10.1.0.0/16`
   - Production DR: `10.2.0.0/16`
3. Create two Availability Zones within the initial VPC.
4. Create public and private subnets in each Availability Zone.
5. Create and configure Internet Gateways for the VPC.
6. Configure NAT Gateways and route tables to provide controlled outbound internet access from private subnets.
7. Configure security groups to restrict communication between public-facing resources and private application resources.
8. Deploy an Application Load Balancer into the public subnets and configure HTTPS listeners.
9. Test connectivity between the public and private network tiers.
10. Test that private resources cannot receive unsolicited inbound internet connections.
11. Refactor the working network configuration into reusable Terraform modules.
12. Deploy the modules into the Development account, Production account, and DR region.
13. Confirm that the three VPCs use non-overlapping CIDR ranges and remain isolated.

**Validation:**

- Run `terraform plan` and confirm that the expected networking resources are created.
- Verify that public subnets have the intended internet routing.
- Verify that private subnet outbound traffic uses the NAT Gateway.
- Test that the public ALB can reach its intended private application targets.
- Attempt unauthorized access to private resources from outside the VPC and confirm that it is blocked.
- Confirm that development resources cannot directly communicate with production resources through unintended network paths.
- Review the resulting Terraform plan for unintended changes.

**Done when:**

- Development, primary production, and DR VPCs are deployed with the intended subnet architecture.
- Routing and security group rules enforce the intended traffic boundaries.
- Public-to-private connectivity works only through authorized paths.
- Network isolation tests produce the expected results.
- Terraform can reproduce the network infrastructure through reusable modules.

---

## Milestone 3: Identity and Encryption

**Goal:** Establish least-privilege identity management, secure CI/CD authentication, and encryption for sensitive infrastructure resources and application data.

**Tasks:**

1. Create the IAM roles required for infrastructure deployment and application access.
2. Define least-privilege IAM policies for each role.
3. Create separate GitHub Actions deployment roles for Development and Production.
4. Configure the GitHub OIDC identity provider in AWS.
5. Create OIDC trust policies restricted to the intended GitHub repository and deployment environment.
6. Configure GitHub Environment protection rules for Production deployments.
7. Configure permission boundaries where appropriate to restrict the maximum permissions that can be granted to deployment identities.
8. Create KMS keys for encrypting supported infrastructure resources and application data.
9. Configure key policies to restrict key administration and usage to authorized principals.
10. Configure encryption for applicable S3 buckets, EBS volumes, and RDS resources.
11. Configure AWS Secrets Manager for application credentials and other sensitive application secrets.
12. Restrict secret retrieval permissions to the application roles that require them.
13. Verify that the Development deployment role cannot assume the Production deployment role.
14. Test authorized and unauthorized IAM operations.

**Validation:**

- Confirm that GitHub Actions can authenticate to AWS through OIDC without stored long-lived AWS access keys.
- Confirm that the Development deployment role can perform its intended operations.
- Attempt a Production operation using the Development role and confirm that it is denied.
- Attempt an unauthorized KMS operation and confirm that access is denied.
- Verify that supported storage resources use the intended KMS encryption configuration.
- Confirm that application secrets can be retrieved by authorized roles but not unrelated identities.

**Done when:**

- Development and Production have separate deployment identities.
- GitHub Actions uses OIDC federation and short-lived credentials.
- IAM permissions follow least privilege.
- KMS encryption is configured for the required resources.
- Secrets Manager stores application secrets with restricted access.
- Authorized and unauthorized access tests produce the expected results.

---

## Milestone 4: Security Monitoring and Governance

**Goal:** Establish centralized security monitoring, organization-wide audit logging, and governance controls across the AWS accounts.

**Tasks:**

1. Configure AWS Organizations and verify that the four accounts are members of the organization.
2. Create the required organizational units and apply appropriate Service Control Policies.
3. Configure organization-wide multi-region CloudTrail logging.
4. Configure the Security-Logging account as the centralized destination for audit logs.
5. Create the centralized S3 logging bucket in the Security-Logging account.
6. Configure S3 bucket policies to allow authorized workload accounts to deliver logs without granting them permission to read or modify existing logs.
7. Enable S3 versioning, encryption, and Object Lock with the appropriate retention configuration.
8. Configure KMS key policies for centralized log encryption and cross-account delivery.
9. Enable VPC Flow Logs for the required VPCs.
10. Configure GuardDuty across the organization and enable the required regional coverage.
11. Configure Security Hub to aggregate security findings in the Security-Logging account.
12. Enable AWS Config in the required accounts and regions.
13. Configure selected AWS Config rules to evaluate infrastructure compliance.
14. Create a controlled security test event or introduce a safe, deliberate misconfiguration.
15. Verify that the relevant service detects the event and that the resulting finding or audit record reaches the expected destination.

**Validation:**

- Generate a test AWS API activity event and verify that it appears in the centralized CloudTrail logs.
- Verify that VPC Flow Logs are delivered to their configured destination.
- Confirm that the Security-Logging account can retain and access the required audit records.
- Attempt to delete or modify protected historical logs using a workload-account identity and confirm that the operation is denied.
- Generate a controlled security finding and verify that GuardDuty or Security Hub reports it.
- Introduce a safe compliance violation and confirm that the relevant AWS Config rule detects it.
- Verify that the Security-Logging account has no unnecessary administrative access into Development or Production.

**Done when:**

- Organization-wide audit logging is operational.
- Security logs are centralized in the dedicated Security-Logging account.
- Historical logs have the intended immutability protections.
- GuardDuty, Security Hub, and AWS Config are configured for the required accounts and regions.
- At least one deliberate security test produces a verifiable finding or logged event.
- The source, destination, and outcome of the test are documented.

---

## Milestone 5: Secure CI/CD Integration

**Goal:** Integrate the existing CDP capstone pipeline with the Terraform infrastructure workflow, ensuring that insecure or invalid infrastructure changes are blocked before deployment.

**Tasks:**

1. Review the existing CDP capstone GitHub Actions pipeline and identify where Terraform deployment steps should integrate.
2. Add Terraform formatting and validation checks to the workflow.
3. Integrate Checkov to scan Terraform configurations for security misconfigurations.
4. Configure the pipeline to generate Terraform plans for proposed infrastructure changes.
5. Make Terraform plan output available for review before deployment.
6. Configure Development deployment using its dedicated OIDC IAM role.
7. Configure Production deployment using its separate OIDC IAM role.
8. Require GitHub Environment approval before Production deployment.
9. Ensure that failed formatting, validation, or security checks prevent deployment.
10. Introduce a deliberately insecure Terraform configuration, such as a publicly accessible S3 bucket or an overly permissive security group.
11. Submit the insecure configuration through a test pull request and verify that Checkov identifies the misconfiguration.
12. Correct the configuration and submit the updated pull request.
13. Verify that the corrected configuration passes the required pipeline checks.

**Validation:**

- Confirm that the pipeline executes Terraform formatting and validation checks.
- Confirm that Checkov identifies the deliberately introduced security misconfiguration.
- Verify that the insecure pull request cannot proceed through the protected deployment workflow.
- Confirm that the corrected configuration passes the pipeline.
- Verify that Development deployment succeeds using its intended IAM role.
- Confirm that Production deployment requires the configured approval.
- Verify that the pipeline cannot assume unrelated AWS roles.

**Done when:**

- The existing CDP pipeline is integrated with Project A's Terraform workflow.
- Terraform validation and Checkov security scanning run automatically.
- Invalid or insecure configurations are blocked before deployment.
- Production deployment requires approval.
- The deliberately insecure configuration is caught, corrected, and successfully revalidated.
- Evidence of the failed and successful pipeline runs is saved.

---

## Milestone 6: Disaster Recovery, Cost Controls, and Final Validation

**Goal:** Demonstrate that the platform can recover from a regional failure, operate within the project's cost constraints, and be reproduced and destroyed through documented Terraform procedures.

**Tasks:**

1. Define measurable Recovery Time Objective (RTO) and Recovery Point Objective (RPO) targets for the platform.
2. Document the expected DR failover sequence, including DNS changes, application scaling, and database promotion.
3. Configure the required cross-region data synchronization for the production data tier.
4. Verify that the DR environment maintains the intended warm-standby configuration.
5. Document the procedure for promoting the DR database to the primary writable database.
6. Simulate a controlled primary-region failure in an isolated test environment.
7. Execute the documented DR failover procedure.
8. Verify that the DR application tier becomes operational and that Route 53 directs traffic to the DR region.
9. Measure the actual recovery time and data recovery point achieved during the test.
10. Document the procedure for recovering from a failed Terraform apply, including how to inspect state, identify partial changes, and reconcile infrastructure using Terraform plans.
11. Test the documented infrastructure recovery procedure in a safe environment.
12. Review the monthly estimated cost of the infrastructure and identify expensive resources.
13. Configure cost monitoring and budget alerts.
14. Verify that the infrastructure can be destroyed through Terraform without leaving unintended billable resources behind.
15. Rebuild the infrastructure from the documented repository and configuration without relying on undocumented manual console changes.
16. Review all acceptance criteria from `PROJECT_CHARTER.md` and confirm that the required evidence has been collected.
17. Update the README, RUNBOOK, SECURITY_DECISIONS, and TEST_RESULTS documentation.

**Validation:**

- Execute the DR test and record the actual recovery time and data recovery point.
- Confirm that the DR database can be promoted and that the application can operate in the secondary region.
- Verify that DNS directs traffic to the intended region after failover.
- Simulate a failed Terraform deployment in a safe environment and follow the documented recovery procedure.
- Review AWS Cost Explorer or equivalent billing information to assess resource costs.
- Execute Terraform teardown and verify that the intended resources are removed.
- Rebuild the infrastructure using the documented Terraform configurations.
- Review the final architecture, threat model, implementation evidence, and acceptance criteria.

**Done when:**

- RTO and RPO targets are documented and tested against an actual recovery exercise.
- The warm-standby DR environment successfully handles a controlled failover.
- A failed Terraform apply can be recovered through a documented and tested procedure.
- Cost monitoring and teardown procedures are operational.
- The platform can be reproduced from code without undocumented console configuration.
- All Project A acceptance criteria have supporting evidence.
- Final documentation is complete and the repository is ready for portfolio review.

---

## Implementation Tracking

Each milestone should be completed and validated before proceeding to the next.

| Milestone | Status |
|---|---|
| 1. Repository and Terraform Foundation | Not Started |
| 2. Network Infrastructure | Not Started |
| 3. Identity and Encryption | Not Started |
| 4. Security Monitoring and Governance | Not Started |
| 5. Secure CI/CD Integration | Not Started |
| 6. Disaster Recovery, Cost Controls, and Final Validation | Not Started |

## Project Completion Criteria

Project A is considered complete when:

- The infrastructure is reproducible through Terraform.
- Development and Production are isolated.
- IAM access follows least-privilege principles.
- Data is encrypted at rest and protected in transit.
- Security monitoring and centralized logging are operational.
- CI/CD prevents insecure infrastructure changes from being deployed.
- At least three attack/failure scenarios are demonstrated and detected:
  1. IAM/access failure.
  2. Network isolation failure.
  3. DR/redundancy failure.
- Recovery and teardown procedures have been tested.
- The architecture and engineering decisions can be explained without relying on the documentation as a script.
