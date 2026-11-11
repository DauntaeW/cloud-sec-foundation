# ARCHITECTURE.md

## 1. Account Structure
(one sentence per account — Management, Security-Logging, Dev, Prod — what each one does)
Management - Root account, handles centralized billing, create member accounts, and manage top level governance policies
Security-Logging - Collects, aggregates, and immutably stores organization-wide security logs, audit trails, and compliance telemetry 
Development - Provides an isolated sandbox for engineers to build test and experiment with new code and features under relaxed controls
Production - Hosts live, consumer facing workloads and enterprise applications with strict access policies, heavy monitoring, and complete isolation from dev changes

## 2. Network Design
(per VPC: CIDR range, AZ count, subnet purpose — don't need exact IPs yet, just the pattern)
2 AZs per VPC
each VPC gets its own non-overlapping /16 (e.g., Dev 10.0.0.0/16, Prod-primary 10.1.0.0/16, Prod-DR 10.2.0.0/16) — non-overlapping matters because if you ever VPC-peer or connect via Transit Gateway, overlapping ranges break routing.
Public subnet purpose is to house public facing application load balancers (ALBs) and NAT gateways. This is the only tier with direct access to the internet
Private subnet purpose is to house your actual application servers, EC2 instances, EKS micrservices, and ECS tasks. Outbout internet traffic goes thought the NAT Gateways in the public tier

## 3. Traffic Flow
(how does a request reach an app? Internet → IGW → ? Explain public vs private subnet roles)
User browser -> DNS route 53 -> VPC via IGW -> Public subnets -> Public ALBs -> Private Subnets
Public and private subnets explaind in #2

## 4. Cross-Account Relationships
(what flows between accounts — logging, CI/CD role assumption — and what does NOT, e.g. "Dev cannot access Prod resources")
(i)Security and logging aggregation - AWS cloudtrail logs, VPC flow logs, and AWS Guarduty findings from dev and prod accounts to the security-logging account. 
Resources in the dev and prod accounts write logs directly to a centralized, immutable amazon S3 bucket located inside the security and logging account. This is
achieved by using bucket policies that grant cross-account s3: PutObject permissions
(ii)CI/CD - IAM role assumptions and deployment metadata. The pipeline assumes a pre-configured IAM deployment role inside the target dev or prod account using AWS security token service sts:AssumeRole
(iii)Organization governance - Management account to all other accounts. What flows? - SCPs. SCPs flow downward creating a guardrail that even the root in dev and prod cannot accounts cannot override.
(iv)Zero trust between environments, dev and prod isolation
(v)No cross environment access via the management account. The management account is not an application proxy, dev data plane traffic never passes through the management account to reach prod
(vi)Log retrieval from dev and prod accounts, while they can write logs to the security account, they have no permissions to read or modify them

## 5. Primary vs DR Region Differences
(what's different about eu-west-1 besides just "it exists" — is it active-passive? does it only activate on failover?)
Active - Passive specifically warm standby. The primary region handles 100% of the live user traffic. ALB is actively receiveing requests, and the auto scaling groups in the private subnets are fully scaled up to handle the production load.
The DR region handles 0% of the live user traffic, the IGW, ALBs and trasit gateways are fully provisioned and sitting warm but route 53 is deliberately sending zero user queries here
To keep costs minimized the application tier is minimized or completey turned off, while the data tier stays synchronized. The network stack is always 100% active, the data tier 100%, but the app tier scaled down to a minimum, with the smallest instance left running jsut to confirm health
If theres an outage and the failover activates, the app tier and compute will activate. DNS switch by route 53 then theres databse promotion, the read-only database in the dr region becomes the new standalone primary writeable database.
Within minutes or even seconds the DR region is fully active and accepting user requests and querying the database.