Objective: A secure, multi-region AWS platform for a cost sensitive startup workload, with dev/prod isolation, least-privilege access, and automated deployment via CI/CD
In Scope: - Primary region with development and production
       - Secondary region with production only
       - VPC architecture: public&private subnets, NAT, SGs, route tables
       - IAM: least privilege roles, permission boundaries
       - KMS encryption at rest - EBS, S3, RDS, Secrets Manager
       - TLS in transit - ALB/NLB HTTPS-only listeners, VPC endpoints, enforced SSL on data stores 
       - Secrets Manager/Parameter Store for app secrets
       - Cloud Trail(org multi region aggregation), GuardDuty, Security Hub, Config rule(compliance as code)
       - Terraform remote state: S3 + DynamoDB lock, per environment state isolation
       - CDP capstone's CI/CD pipeline deploys into this platform
       - Misconfigurations deliberately introduced
Out of Scope: - EKS runtime hardening(I am not securing Kubernetes)
       - I am not securing the software supply chain
       - This wll not support continuous security loop(SOAR)
Budget/Teardown - This is a project that will simulate a startup company's infrastructure - cost sensitive, $100/month while running, infra is spun up for demo/teardown-tested then destroyed, in production they expect steady volume, and data is to be deleted if not accessed after 6 months
Acceptance criteria - Every engineering decision can be explained out loud, unscripted
       - Architecture diagram made and accurate
       - Threat model doc written before build started
       - At least 3 attack/failure demonstrated and caught: one IAM/access failure, one network isolation test, one DR/redundancy test 
       - Reproducible from code only