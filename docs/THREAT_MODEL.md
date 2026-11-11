# THREAT_MODEL.md

## 1. Assets
(what are we protecting? list concretely: Terraform state, IAM credentials, 
app data in RDS, CloudTrail logs, KMS keys, CI/CD OIDC token)
App data - PII, payment tokens and customer IDs
Logs - CloudTrail (API activity — who did what, when), VPC Flow Logs (network traffic metadata),Application logs (your app’s own output — could contain PII if not scrubbed, worth noting as a risk itself)
Secrets/Credentials - Database credentials (RDS master password, app DB user), API keys (Stripe secret key, any third-party integrations), IAM access keys (should be minimal/none if using OIDC correctly — good thing to note)
IAM roles/policies themselves — not just the secrets, but the permission definitions. If an attacker edits a policy (not steals a key), they can grant themselves access without ever touching a credential.
Terraform state file — this is huge and commonly missed. State contains a full map of your infrastructure, often including plaintext secrets, resource IDs, IPs. If someone reads it, they have a blueprint of your entire platform. If someone writes to it, they can make Terraform think reality is different than it is (dangerous — can cause Terraform to destroy/recreate things).
KMS keys — the encryption keys themselves. If these are compromised, everything encrypted with them is compromised too — doesn’t matter how good your encryption is if the key leaks.
CI/CD pipeline / OIDC trust relationship —  If attacker compromises this, they don’t need your AWS credentials at all — they just push malicious code and your own pipeline deploys it for them.
Availability of the platform itself — Availability as an asset. DoS isn’t just “annoying,” it’s a business-continuity asset worth naming.
AWS account access itself (root/management account credentials) — the nuclear option. Full account takeover.

## 2. Trust Boundaries
(where does trust level change? e.g. "Internet → Public Subnet" is a boundary,
"Dev account → Prod account" is a boundary, "GitHub Actions → AWS" is a boundary)
Dev -> Prod
Github Actions -> AWS
Internet -> Public Subnet
Public Subnet -> Private Subnet
Dev/Prod → Security-Logging (write-only, one-directional — this is fine, low risk)
Security-Logging → Dev/Prod (should not exist at all — Security-Logging has no reason to reach back into workload accounts)
Management Account -> All other accounts
Private Subnet service → Private Subnet service (east-west)

## 3. Threats (STRIDE per boundary)
(for each trust boundary above, list 1-2 realistic threats using STRIDE categories)
Boundary: 
- GitHub Actions → AWS
Threat 1: [S] Spoofing — attacker-controlled PR workflow spoofs a legitimate 
          deploy run, tricking AWS into trusting a malicious OIDC token
Threat 2: [E] Elevation of Privilege — loosely-scoped trust policy lets a 
          dev-only workflow assume the prod deployment role
- Dev -> Prod
Threat 1: [T] Tampering - Unauthorised modifications to code, confirgurations, or binaries as they move through the CI/CD pipeline from dev to prod. An attacker could alter container image or build artifact, modifying a terraform script
Threat 2: [E] Elevation of privilege - Lower trust dev environment gains unauthorized, higher level access to the prod environment. Utilizing overl permissive IAM roles to deploy directly to production, bypassing standard approvals
- Internet -> Public Subnet
Threat 1: [S] Spoofing - spoofed identity via invalid/forged TLS client certs.
Threat 2: [D] Denial of service - Floods public endpoints like an IGW, or API requests, with massive volume of traffic to either knock applications offline or strain budget.
- Public Subnet -> Private Subnet
Threat 1: [I] Information disclosure - The main objective is preventing an attacker who has compromised a public resource from moving laterally. for exmaple a compromised public facing resource like an EC2 instance running a SSRF attack to scrape internal database records, or unencrypted data packets being interpreted between subnets.
Threat 2: [E] Elevation of privilege - An attacker leverages a vulnerability in a public service like a RCE bug on a public web server to execute commands that exploit misconfigured network or IAM permissions, granting them control over private critical infrastructure
- Dev/Prod → Security-Logging
Threat 1: [T] Tampering - A compromised application attempts to delete alter or inject fake log entries into the logging destination to hide unauthorized activities
Threat 2: [I] Information Disclosure - Highly confidential data accidentally leaks into the log stream in plain text, at the source, or an unauthorized entity finds a way to read or intercept the log data en route to the logging bucket
- Security-Logging → Dev/Prod
Threat 1: [E] Elevation of privilege - A compromised centralized security tool using an overly permissive cross account IAM role to execute commands, or deploy infra back into a prod account
Threat 2: [I] Information Disclosure - A reverse connection allowing data to leak backward from the secure aggregation into less restricted environments, exposing sensitive security configurations, global network topologies, or active incident response data to unauthorized developers or compromised workloads
- Management Account -> All other accounts
Threat 1: [E] Elevation of privilege - An attacker compromising a single identity or application within the management account and leverages the inherent trust relationship to gain root or admin control over every single member account in the organization
Threat 2: [T] Tampering - An attacker or malicious insider uses the management account to alter or delete organizational security controls leaving all other accounts exposed and vulnerable.
- Private Subnet service -> Private Subnet Service (east-west)
Threat 1: [S] Spoofing - A compromised service or container laterally impersonates another legitimate internal service to gain access to peer resources or databases for example compromising a private web api and spoofing internal service discovery/dns entries, or manipulating unsecured HTTP headers to ptricka  peer microservice into processing fraudulent requests.
Threat 2: [I] Information Disclosure - If an attacker compromises a single container or instance within the private subnet, they can sniff local network traffic or query unauthorized peer endpoints to harvest sensitive data. For example running a network packet capture on a compromised private container to interecept plain text database credentials, API tokens, or unencrypted PII passing between backend services

## 4. Mitigations
(for each threat, name the specific control - not "we'll secure it" but
"OIDC federation + scoped IAM role restricts GitHub Actions to deploy-only permissions")
GitHub Actions → AWS

• Threat 1 [S]: OIDC trust policy scoped to repo/ref only
	• [IMPLEMENT] — Zero cost, essential for basic platform safety, and directly supports the charter's requirement for automated deployment via CI/CD.
• Threat 2 [E]: Separate dev/prod IAM roles + GitHub Environment protection
	• [IMPLEMENT] — Necessary to achieve the charter's explicit requirement of dev/prod isolation and least-privilege access.

Dev → Prod

• Threat 1 [T]: Cryptographic artifact signing, immutable image SHAs, IaC linting
	• [DEFER] — The charter explicitly states: "I am not securing the software supply chain" as Out of Scope. While IaC linting can be implemented easily, full cryptographic artifact signing should be deferred to a later phase.
• Threat 2 [E]: Short-lived OIDC tokens, multi-party approvals, separate cloud accounts
	• [IMPLEMENT] — Fits perfectly with the "dev/prod isolation" and "Terraform per-environment state isolation" requirements in the charter.

Internet → Public Subnet

• Threat 1 [S]: ALB with enforced mTLS / PKI infrastructure
	• [DEFER] / [ACCEPTED RISK] — As your note correctly calls out, public users do not have certificates. Deploying a private CA and enforcing mTLS here breaks public accessibility. Instead, IMPLEMENT server-side HTTPS via ACM + WAF.
• Threat 2 [D]: AWS Shield Standard, Auto-scaling, WAF rate limiting
	• [IMPLEMENT] — Shield Standard is free. Basic WAF rate limiting and aggressive auto-scaling thresholds protect your cost-sensitive budget ($100/mo) from being blown out by a simple script, aligning with the charter.

Public Subnet → Private Subnet

• Threat 1 [I]: Security Group chaining, E2E TLS, VPC flow logs + anomaly detection
	• [IMPLEMENT] (Partial) — Security Group chaining and TLS in transit are core architecture requirements in your charter. However, DEFER advanced anomaly detection on VPC Flow logs, as it requires paid services (like GuardDuty/CloudWatch advanced features) that could easily jeopardize your $100/mo limit during demo testing.
### You deferred “GuardDuty/CloudWatch advanced anomaly detection” under Public Subnet→Private Subnet for budget reasons — but your charter’s In Scope list explicitly includes GuardDuty as a required service. Check: GuardDuty’s standard tier is pay-per-event-analyzed, typically a few dollars/month at low/demo traffic volumes — it’s not the expensive part. What’s expensive is the advanced correlation/anomaly features layered on top. So don’t defer GuardDuty itself (charter requires it) — just defer the custom anomaly-detection tooling built on top of its findings. Reword that one line so it doesn’t contradict your own charter.
• Threat 2 [E]: Least privilege instance profiles, network micro-segmentation
	• [IMPLEMENT] — Directly matches your charter requirements ("VPC architecture: SGs", "IAM least privilege roles"). It is also a perfect setup for your required network isolation test.

Dev/Prod → Security-Logging

• Threat 1 [T]: S3 Object Lock (WORM), dedicated log account, write-only IAM
	• [IMPLEMENT] — Matches the "CloudTrail org multi-region aggregation" objective and guarantees tamper-proof logs for security baselines without high overhead.
• Threat 2 [I]: Automated log-scrubbing pipelines, KMS CMK separation
	• [DEFER] — Log-scrubbing pipelines involve custom code/lambdas or heavy pattern matching. Given your $100/mo budget and quick demo/teardown cycle, this is too complex for Project A. IMPLEMENT the KMS separation, but DEFER the automated scrubbing pipeline.

Security-Logging → Dev/Prod

• Threat 1 [E]: SCPs denying cross-account role assumption from log account
	• [IMPLEMENT] — Implemented via AWS Organizations for free. Directly ensures that a compromise of the logging aggregate cannot pivot backwards.
• Threat 2 [I]: KMS Key policies restricting external principals, MFA-delete
	• [IMPLEMENT] — Standard KMS policy design that fits within the infrastructure-as-code deployment and costs nothing extra.

Management Account → All other accounts

• Threat 1 [E]: Move workloads out of management account, break-glass MFA
	• [DEFER] — Standard landing zone best practice. The management account should only host the AWS Org and billing structures, keeping your workload accounts isolated.
### note it as “documented in RUNBOOK.md as a future operational control” instead. What you CAN and should implement now is simpler: MFA required on your own IAM user for any privileged action, which is a one-line IAM condition, zero cost, no extra process. Split the threat into those two pieces.
• Threat 2 [T]: Immutable Org-level SCPs, independent core infra pipelines
	• [IMPLEMENT] — Enforces the structural integrity of your platform from day one via Terraform.

Private Subnet service → Private Subnet Service (East-West)

• Threat 1 [S]: Mutual TLS via Service Mesh, Private CA, SPIFFE/SPIRE identifiers
	• [DEFER] — As your text note brilliantly states, your charter explicitly leaves "EKS runtime hardening" Out of Scope. A full service mesh + SPIRE setup is a multi-week enterprise endeavor that will blow past a $100/mo budget and conflict with your "quick teardown/reproducible from code only" timeline.
• Threat 2 [I]: Application layer encryption, granular SGs, basic K8s NetworkPolicies
	• [IMPLEMENT] — This is the realistic alternative for your current project. Use basic cloud security groups scoped to specific ports/sources and basic native Kubernetes NetworkPolicies to manage container isolation simply.






## 5. Residual Risk
(what risk remains even after mitigation — nothing is 100% secure, say so honestly)
- GitHub Actions → AWS
Threat 1: [S] Spoofing - If GitHub's own OIDC signing infra is compromised, this control fails 
          (out of your control — noted for Section 5)
Threat 2: [E] Elevation of Privilege - If an attacker gains administrative access to the GitHub repository, they can bypass environment protections or rewrite the workflow to target production variables. Residual risk also persists if an authorized approver inadvertently approves a malicious PR targeting the production environment.
- Dev -> Prod
Threat 1: [T] Tampering - Pipeline Compromise. If an attacker compromises the underlying CI/CD platform, they can sign malicious artifacts at the source before deployment
Threat 2: [E] Elevation of privilege - Token Theft. Valid, short-lived tokens or active developer pipeline sessions can still be exfiltrated and abused during their brief window of validity.
- Internet -> Public Subnet
Threat 1: [S] Spoofing - Zero day protocol Exploits. Vulnerabilities in the underlying cryptographic libraries could allow attackers to bypass certificate validation
Threat 2: [D] Denial of service - Economic Denial of Sustainability. While auto-scaling keeps the application online, a prolonged, sophistcated application layer attack can result in astronomical cloud billing before automated throttles trigger
- Public Subnet -> Private Subnet
Threat 1: [I] Information disclosure - Application layer exfiltration. if a public app ishit with an SSRF attack, it acts as a legitimate proxy. The network layers see authorized traffic, but the attacker is actively reading data through the app
Threat 2: [E] Elevation of privilege - unpatched OS/Software vulnerabilities. An attacker exploiting a remote code execution bug inside a heavily restricted app can still move laterally if a kernel exploit exists on adjacent private operating systems
- Dev/Prod → Security-Logging
Threat 1: [T] Tampering - Log Suppression/Starvation. While an attacker cannot alter historical logs, a compromised admin account could kill the log-forwarding agent at the source, creating a blind spot moving forward.
Threat 2: [I] Information Disclosure - Scrubbing Bypass. Highly unusual or non-standard token patterns, custom-encoded passwords, or nested JSON structures might slip past regex patterns and leak into the datastore
### Dev/Prod→Security-Logging, Threat 2 residual (Scrubbing Bypass) — slight logical mismatch. You tagged the scrubbing pipeline mitigation as [DEFER] in Section 4 — meaning it’s not built at all for Project A. If it’s not built, “bypass” isn’t quite the right framing (nothing to bypass). Reframe as: “Accepted risk — without an automated scrubbing pipeline, PII/secrets accidentally logged by the application remain in plaintext logs until a future phase implements scrubbing.” Same risk, more honest about why it exists (deferred scope, not a flawed control).
- Security-Logging → Dev/Prod
Threat 1: [E] Elevation of privilege - Shared infrastructure compromise. If an organization-wide admin identity in compromised, the attacker can alter the network/IAM topology to explicitly create the missing path
Threat 2: [I] Information Disclosure - Malicious Security insider. A rogue engineer who ligitimately holds admin rights within the central security account could manually exfiltrate the aggregated data to an external, public internet endpoint
- Management Account -> All other accounts
Threat 1: [E] Elevation of privilege - Identity provider compromise. If your central corporate IDp is entirely compromised, an attacker can forge assertions to bypass cloud level MFA and claim admin tokens.
Threat 2: [T] Tampering - Root account credentials. The absolute root email/password of the management account bypasses all SCPs. If the physical hardware tokens or root credentials are physically or socially engineered, the entire organization is compromised.
### Management Account, Threat 1 residual (IdP compromise) — you haven’t established anywhere in your docs that there’s a corporate identity provider (Okta/Azure AD) federating into AWS. If your actual login path is AWS IAM Identity Center or just IAM users directly, this residual risk doesn’t apply to your architecture as designed. Either add “external IdP federation” to your ARCHITECTURE.md if that’s genuinely part of the design, or swap this residual for what’s actually true for you: root credential compromise (which you already correctly cover in Threat 2) is really your single point of failure here — Threat 1 and 2 overlap more than they’re distinct. Consider merging or replacing Threat 1 with something like “if IAM Identity Center or SSO session tokens are stolen, an attacker gains the same access as the legitimate user within that token’s lifetime.”
- Private Subnet service -> Private Subnet Service (east-west)
Threat 1: [S] Spoofing - Sidecar/agent compromise. if an attacker achieves root access on an instance or container, they can bypass or subvert the local sidecar proxy to intercept token or directly make calls using the local machine's identity.
Threat 2: [I] Information Disclosure - Application layer logic flaws/bypasses. Even with encrypted pipelines and strict network rules, if an authorized peer service is vulnerable to an injection flaw, an attacker can abuse the legitimate network path to trick that peer into returning sensitive database records.