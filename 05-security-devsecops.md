# Security & DevSecOps — Interview Questions

> Grounded in the zen-pharma security stack: CodeQL, Semgrep, Trivy, OWASP Dep Check, Cosign/Sigstore, Kyverno, GitHub OIDC, ESO + IRSA.
> 25 questions covering container security, SAST/SCA, supply chain, IAM, DevSecOps philosophy, and TLS/certificate management.

---

## Table of Contents

1. [Container Security](#1-container-security) — Q1–Q4
2. [SAST & Code Scanning](#2-sast--code-scanning) — Q5–Q8
3. [Dependency & Image Scanning](#3-dependency--image-scanning) — Q9–Q11
4. [Supply Chain Security](#4-supply-chain-security) — Q12–Q14
5. [AWS IAM & OIDC](#5-aws-iam--oidc) — Q15–Q19
6. [DevSecOps Philosophy](#6-devsecops-philosophy) — Q21–Q23
7. [External Secrets Operator](#7-external-secrets-operator) — C2–C3
8. [TLS/SSL Certificate Management](#8-tlsssl-certificate-management) — C4

---

## 1. Container Security

---

**Q1: What is a multi-stage Docker build and why does it matter for security?**

- **What is being tested:** Container image hygiene.
- **Strong answer:** A multi-stage build uses multiple `FROM` instructions. Early stages contain build tools (Maven, JDK SDK, npm). The final stage copies only the compiled artifact from the build stage — no Maven, no JDK, no source code, no node_modules dev dependencies make it into the production image. This reduces image size (smaller attack surface), removes tools that have no business in a runtime container (Maven could be used to download arbitrary JARs if compromised), and reduces the number of CVEs Trivy finds because fewer packages are present.

**Sample Dockerfile (Spring Boot / Maven, matching this project's stack):**

```dockerfile
# ---- Stage 1: build ----
FROM eclipse-temurin:17-jdk-alpine AS build
WORKDIR /app

# Copy only the POM first so Docker can cache the dependency layer
# separately from source changes — deps only re-download when the POM changes
COPY pom.xml .
COPY .mvn/ .mvn/
COPY mvnw .
RUN ./mvnw dependency:go-offline -B

# Now copy source and build — this layer invalidates on every code change,
# but the dependency layer above stays cached
COPY src ./src
RUN ./mvnw clean package -DskipTests -B

# ---- Stage 2: runtime ----
FROM eclipse-temurin:17-jre-alpine AS runtime
WORKDIR /app

# Non-root user — see Q2. Alpine's `addgroup`/`adduser`, not useradd/groupadd
ARG UID=1000
ARG GID=1000
RUN addgroup -g ${GID} appgroup && \
    adduser -D -u ${UID} -G appgroup appuser

# Only the compiled JAR crosses the stage boundary — no Maven, no JDK,
# no source code, no ~/.m2 cache make it into this image
COPY --from=build --chown=appuser:appgroup /app/target/*.jar app.jar

USER appuser
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

Why each piece matters for security specifically, not just image size:
- **`eclipse-temurin:17-jdk-alpine` never reaches the final image** — the full JDK includes `jshell`, `jdb`, `javac`, and other tooling an attacker could use to compile and run arbitrary code inside a compromised container. The runtime stage only has the JRE, which can *execute* bytecode but not compile or debug it.
- **`COPY --from=build` is the actual security boundary** — it's a deliberate allowlist (only `/app/target/*.jar` crosses over) rather than a denylist of things to remove after the fact. Nothing from the build stage exists in the final image unless this line explicitly copies it.
- **`--chown=appuser:appgroup` on the COPY, plus `USER appuser` before `ENTRYPOINT`** — the JAR is owned by and runs as UID 1000, not root, tying directly into the Q2 answer about limiting blast radius on container escape.
- **Splitting `COPY pom.xml` from `COPY src`** is a build-cache optimization, not a security control, but it's worth knowing: without it, every source change invalidates the dependency-download layer too, making iterative builds far slower — which is a big part of *why* teams reach for multi-stage builds like this instead of skipping the caching step.

---

**Q2: Why should containers never run as root?**

- **What is being tested:** Container escape and privilege escalation awareness.
- **Strong answer:** If a container runs as root (UID 0) and a container escape vulnerability is exploited, the attacker gets root on the host node. From there they can read secrets mounted to other pods, modify host-level configurations, or pivot to the Kubernetes control plane. Running as UID 1000 limits the blast radius — the attacker gets the privileges of a non-privileged user. Kubernetes PodSecurityAdmission and tools like Kyverno can enforce `runAsNonRoot: true` at admission time, rejecting any pod spec that attempts to run as root.
- **Project context:** Our Dockerfiles use `--build-arg UID=1000 --build-arg GID=1000` and set `USER 1000` in the final stage.

---

**Q3: What is the difference between a Docker image tag and a digest?**

- **What is being tested:** Image immutability understanding.
- **Strong answer:** A tag (e.g. `myimage:sha-abc1234`) is a mutable pointer — it can be reassigned to point to a different image by pushing a new image with the same tag. A digest (e.g. `sha256:abc123...`) is the cryptographic SHA256 hash of the image manifest — it is content-addressable and immutable. The same digest always refers to the exact same bytes. For security-sensitive deployments, you reference images by digest, not by tag, in production manifests. Cosign signs the digest, not the tag, for this reason.

---

**Q4: How do you reduce the size of a Docker image?**

- **What is being tested:** Practical Docker knowledge.
- **Strong answer:** (1) Multi-stage builds — only copy what the runtime needs, (2) Use minimal base images — `eclipse-temurin:17-jre-alpine` instead of the full JDK image, (3) Combine `RUN` instructions to reduce layers, (4) Use `.dockerignore` to exclude source files, test reports, and build artifacts from the build context, (5) Remove package manager caches in the same `RUN` instruction that installs them (`apt-get clean && rm -rf /var/lib/apt/lists/*`), (6) Don't install debugging tools (curl, wget, bash) in production images.

---

## 2. SAST & Code Scanning

---

**Q5: What is SAST and at what stage of the pipeline should it run?**

- **What is being tested:** Shift-left security understanding.
- **Strong answer:** Static Application Security Testing analyses source code without executing it, looking for vulnerability patterns. It should run as early as possible — ideally on every push to a feature branch, not just before merging to main. Finding a SQL injection on a feature branch takes minutes to fix. Finding it after merging to develop, building an image, and deploying to DEV takes hours. The cost of fixing a security issue grows exponentially the later it is found in the SDLC.

---

**Q6: What is the difference between CodeQL and Semgrep? Why would you run both?**

- **What is being tested:** Tool depth and understanding of their different approaches.
- **Strong answer:** CodeQL performs deep semantic analysis. It instruments the compilation to build a full call graph and data flow model, then queries it using a custom query language (QL). It excels at finding complex multi-step vulnerabilities — SQL injection where user input flows through three service layers before reaching a query, path traversal assembled across multiple methods. Semgrep uses pattern matching on the AST — it is faster but cannot track data flow across method boundaries. Semgrep excels at framework-specific misconfigurations: Spring Boot CSRF disabled, actuator endpoints without authentication, missing `@PreAuthorize` annotations, CORS wildcards. Running both gives you deep data flow analysis from CodeQL and framework-specific pattern coverage from Semgrep with minimal overlap.

---

**Q7: CodeQL instruments the build. What does that mean and why does it matter?**

- **What is being tested:** Deep CodeQL knowledge.
- **Strong answer:** CodeQL uses a technique called build tracing — it intercepts calls to the compiler (`javac`, `kotlinc`) during the actual `mvn compile` phase to extract type information, method signatures, and call relationships that are only available during compilation. This is why `codeql-action/init` must run before Maven — if you initialise CodeQL after Maven has already compiled the code, the tracer was never in place and CodeQL either produces a weaker analysis or fails with an empty database error. This instrumentation is what enables CodeQL to track data flow across file and class boundaries.

---

**Q8: What are SARIF files and why does your pipeline upload them to GitHub?**

- **What is being tested:** Security tooling integration knowledge.
- **Strong answer:** SARIF (Static Analysis Results Interchange Format) is a JSON-based standard for representing static analysis results — findings, severity, file location, and remediation guidance. GitHub's Code Scanning feature ingests SARIF files and displays findings in the Security tab, aggregated across all tools and services. This means security engineers get a single dashboard showing CodeQL, Semgrep, and Trivy findings across all 7 services without reading raw pipeline logs. Findings can be triaged, dismissed with a reason, and tracked over time.

---

## 3. Dependency & Image Scanning

---

**Q9: What is SCA and how is it different from SAST?**

- **What is being tested:** Knowing the distinction between code vulnerabilities and dependency vulnerabilities.
- **Strong answer:** SAST analyses code you wrote for security vulnerabilities in your logic. SCA (Software Composition Analysis) analyses the third-party libraries and frameworks you depend on for known CVEs published in vulnerability databases like the NVD. You could write perfectly secure code but depend on a version of Spring Boot with Log4Shell — SAST would not catch this because it's not in your code, but SCA would. OWASP Dependency Check and Snyk are SCA tools. Both SAST and SCA are needed.

---

**Q10: What is CVSS and what score threshold makes sense for a build gate?**

- **What is being tested:** Security scoring knowledge and pragmatic thinking.
- **Strong answer:** CVSS (Common Vulnerability Scoring System) is a 0–10 numerical score representing the severity of a vulnerability. 0–3.9 is Low, 4.0–6.9 is Medium, 7.0–8.9 is High, 9.0–10 is Critical. Most security frameworks (PCI-DSS, SOC 2, HIPAA) require remediating High and Critical findings. Setting the build gate at CVSS ≥ 7.0 (High) is a common industry standard — it blocks the build when there is something genuinely exploitable and actionable. Setting it at 9.0 only catches the very worst cases and misses a large class of exploitable vulnerabilities. Setting it at 4.0 blocks too many builds on Medium findings that are often theoretical in your context.

---

**Q11: What is the difference between OWASP Dependency Check and Trivy?**

- **What is being tested:** Understanding the different scanning surfaces.
- **Strong answer:** OWASP Dependency Check scans your declared source dependencies — what's listed in `pom.xml` or `package.json` — against the NVD. It runs before Docker build and catches vulnerable libraries in your application code. Trivy scans the built container image — OS-level packages installed by the Dockerfile base image, the JRE runtime, and all native libraries bundled inside the image. A vulnerable `libssl` in the Alpine base layer or a vulnerable `glibc` would only be caught by Trivy. A vulnerable Spring Boot version would be caught by both but OWASP Dep Check catches it earlier. Both are needed because they scan different surfaces.

---

## 4. Supply Chain Security

---

**Q12: What is supply chain security in the context of container images?**

- **What is being tested:** Whether you understand attacks beyond code vulnerabilities.
- **Strong answer:** Supply chain attacks target the build and distribution pipeline rather than the application code itself. A classic example: SolarWinds — attackers compromised the build system, not the source code. For containers, the threats include: a malicious dependency injected via a compromised package registry, a rogue image pushed directly to ECR bypassing CI, a developer manually pushing a backdoored image. Supply chain security controls include: signing images at build time (Cosign), verifying signatures at deploy time (Kyverno), generating SBOMs to know exactly what's in an image, and pinning dependencies to specific digest-verified versions.

---

**Q13: How does Cosign keyless signing work?**

- **What is being tested:** Sigstore ecosystem knowledge.
- **Strong answer:** Keyless signing uses the workflow's OIDC identity instead of a long-lived private key. The process: (1) GitHub Actions generates an OIDC token that cryptographically identifies the workflow run and repository, (2) Cosign sends this token to Fulcio, a certificate authority in the Sigstore ecosystem, (3) Fulcio verifies the OIDC token and issues a short-lived X.509 certificate (valid for ~10 minutes) that encodes the workflow identity, (4) Cosign signs the image digest using this certificate, (5) the signature and certificate are written to the Rekor public transparency log — a tamper-evident, append-only ledger. There are no long-lived signing keys to manage, rotate, or leak.

---

**Q14: How does Kyverno enforce image signing at the Kubernetes admission level?**

- **What is being tested:** End-to-end supply chain enforcement knowledge.
- **Strong answer:** Kyverno is a Kubernetes-native policy engine that runs as an admission webhook. When a pod is submitted to the API server, the request passes through Kyverno before being accepted. A Kyverno `ClusterPolicy` with `verifyImages` rules instructs Kyverno to check that the image has a valid Cosign signature in Rekor, signed by a specific identity (e.g. the GitHub Actions OIDC identity for your repo). If the signature is missing or invalid, Kyverno rejects the pod with an admission error — the pod never starts. This means even if someone pushes a rogue image directly to ECR, it cannot run in the cluster.

---

## 5. AWS IAM & OIDC

---

**Q15: Why is OIDC-based authentication preferred over long-lived IAM access keys for CI/CD?**

- **What is being tested:** Cloud security fundamentals.
- **Strong answer:** Long-lived access keys have several problems: they don't expire, so a leaked key provides persistent access until manually rotated; they are often accidentally committed to source code or exposed in build logs; rotation is a manual operational burden; they need to be stored as secrets and managed. OIDC eliminates all of these: credentials are generated per workflow run, are scoped to that run only, expire when the run ends (~15 minutes), and there is nothing to store, rotate, or leak. The trust relationship is established cryptographically via the OIDC provider — the IAM role trusts JWT tokens signed by GitHub's OIDC provider for a specific repository and branch.

---

**Q16: How do you scope an IAM role to be as least-privileged as possible for ECR push?**

- **What is being tested:** IAM security knowledge.
- **Strong answer:** The IAM role for CI should have exactly the permissions needed for ECR push and nothing else: `ecr:GetAuthorizationToken` (to get a login token), `ecr:BatchCheckLayerAvailability`, `ecr:PutImage`, `ecr:InitiateLayerUpload`, `ecr:UploadLayerPart`, `ecr:CompleteLayerUpload`. These should be scoped to specific ECR repository ARNs, not `*`. The trust policy should specify the exact GitHub repository and optionally the branch using the `sub` claim condition — e.g. `repo:myorg/myrepo:ref:refs/heads/develop` — so even if another repository somehow obtained a GitHub OIDC token, it could not assume this role.

---

**Q17: What is the `sub` claim in a GitHub OIDC token and why does it matter?**

- **What is being tested:** OIDC token structure knowledge.
- **Strong answer:** The `sub` (subject) claim identifies the entity requesting the token. For GitHub Actions it takes the form `repo:<owner>/<repo>:ref:refs/heads/<branch>` or `repo:<owner>/<repo>:environment:<env>`. The IAM trust policy `StringEquals` condition on the `sub` claim is what prevents other repositories or branches from assuming the role. Without scoping the `sub` claim, any workflow in any repository could assume the role as long as it uses GitHub's OIDC provider. Always lock down the `sub` claim to the specific repo and optionally branch or environment.

---

**Q18: A security audit flags that your RDS password is visible in the Terraform state file. How do you respond?**

<details>
<summary>Expected answer</summary>

**The actual fix: stop passing a password into Terraform at all.** Use RDS-managed master password integration with Secrets Manager:

```hcl
resource "aws_db_instance" "this" {
  manage_master_user_password = true
  # no password argument
}
```

AWS generates and rotates the password directly in Secrets Manager. Terraform state only ever holds a reference (the secret ARN), never the plaintext — so there's nothing in state to leak. This replaces the `db_password` variable in `modules/rds/variables.tf` entirely.

Why marking the variable `sensitive = true` isn't enough: that only suppresses the value from logs and plan/apply output — **it does not encrypt or exclude it from the state file**. Anyone with read access to the state (S3 object, local file, etc.) can still extract the plaintext password with `terraform show` or `jq` on the state JSON.

Defense-in-depth already in place in zen-pharma, for cases where a plaintext secret still ends up in state (e.g. other sensitive resources):

1. **S3 state backend with encryption** — the bucket was created with `AES256` server-side encryption and public access blocked. The state file at rest is encrypted.

2. **S3 versioning** — if the state file is compromised, earlier versions without the secret can be restored.

3. **ESO decouples apps from the master password** — applications never use the RDS master password directly. They connect using credentials from Secrets Manager, which ESO syncs into K8s. Even if the master password did leak from state, it can be rotated without touching application code.

**How zen-pharma actually implements this today:** the password is never written to a `.tfvars` file. It's passed at pipeline runtime from a GitHub Secret:
```yaml
terraform plan -var="db_password=${{ secrets.DEV_DB_PASSWORD }}"
```
The variable is `sensitive = true`, so it's redacted from plan/apply logs. The same value is also written to AWS Secrets Manager (`/pharma/<env>/db-credentials`), and ESO syncs it into K8s as a Secret for the application to consume — the app never reads it from Terraform output. This is "good enough" for the current setup, but as Q18 above shows, the plaintext still lands in the state file, which is exactly the gap `manage_master_user_password = true` closes.

**Variations seen in real enterprise projects**, roughly in order of maturity:

1. **Static password via CI secret (what zen-pharma does)** — simple, but the password is a long-lived static value that still ends up in state, and rotating it means a Terraform apply plus a CI secret update.

2. **`manage_master_user_password = true` (AWS-native)** — the fix recommended above. No password variable, AWS handles generation and rotation, state only holds a secret ARN. Increasingly the default in newer enterprise Terraform modules.

3. **RDS IAM database authentication** — no password at all. The DB user authenticates with a short-lived (15-minute) IAM auth token generated via `rds-db:connect`, requested through the AWS SDK/driver at connection time. Common in shops that want to eliminate credential rotation entirely, at the cost of needing IAM-auth-aware DB drivers and slightly higher connection overhead.

4. **HashiCorp Vault dynamic secrets engine** — Vault owns a privileged DB user and mints a unique, short-TTL username/password pair per consumer on demand (via the `database` secrets engine). Nothing static exists; if a lease isn't renewed it expires and Vault revokes it. Common in large enterprises already standardized on Vault for secrets, since it also covers non-RDS secrets uniformly.

5. **Secrets Manager rotation Lambda** — password is still static-ish but AWS Secrets Manager runs a scheduled Lambda (`AWSSecretsManagerRDSPostgreSQLRotationSingleUser` or custom) that rotates it every N days and updates RDS in place. Middle ground between (1) and (3)/(4) — keeps a traditional password/driver but removes the "never rotated" problem.

6. **SOPS/KMS-encrypted tfvars checked into git** — some enterprises still commit `terraform.tfvars`, but encrypted with `sops` using a KMS key, decrypted only in CI. Avoids plaintext secrets in git history while keeping the "secrets live next to code" workflow some teams prefer over external secret stores.

The interview signal here: naming *why* each variation exists (rotation burden, blast radius of a leaked static credential, driver compatibility) matters more than listing them — plenty of real production systems still use option 1, and that's a defensible choice if the compensating controls (state encryption, ESO, restricted state bucket access) are in place.

</details>

---

**Q19: You need to give a new team member `read-only` access to see what Terraform would change, but not be able to apply. How do you set this up?**

<details>
<summary>Expected answer</summary>

Two parts:

**AWS side** — Create a new IAM policy with only read permissions:
```json
{
  "Action": ["ec2:Describe*", "eks:Describe*", "rds:Describe*", "ecr:Describe*", "s3:GetObject", "s3:ListBucket"],
  "Effect": "Allow",
  "Resource": "*"
}
```
Attach it to the new team member's IAM user. They can run `terraform plan` (which only reads from AWS) but `terraform apply` will fail on any write action.

**GitHub side** — The GitHub Environment (`dev`) has Required Reviewers. The new member can view the Actions tab and see plan output on every PR. They cannot approve the apply job unless added as a reviewer.

**Practically** — the easiest approach is to give them repository read access on GitHub and the read-only IAM policy. They can clone the repo, run `terraform plan` locally, see the output, and comment on PRs — but they cannot merge to `main` or approve the apply gate.

</details>

---

## 6. DevSecOps Philosophy

---

**Q21: What does "shift-left security" mean?**

- **What is being tested:** Core DevSecOps philosophy.
- **Strong answer:** In a traditional SDLC, security testing happened at the end — a penetration test or security review before release. Problems found at that stage are expensive to fix: the code is already written, deployed to staging, and reviewed. Shift-left means moving security checks earlier in the development process — ideally to the developer's own machine or at minimum to the first CI run on a feature branch. The earlier a security issue is found, the cheaper and faster it is to fix. A SAST finding on a feature branch takes minutes to fix. The same finding after merging to main, building an image, and deploying to staging takes hours. After reaching production, it may require a CVE disclosure, customer notification, and emergency hotfix.

---

**Q22: What is an SBOM and why is it increasingly required?**

- **What is being tested:** Supply chain security awareness, compliance trends.
- **Strong answer:** An SBOM (Software Bill of Materials) is a machine-readable inventory of all components in a software artifact — every library, version, and license. It is the equivalent of an ingredient list for software. When a new CVE is disclosed (like Log4Shell), organisations with an SBOM can immediately query it to find which services are affected. Without one, they have to manually audit every service. US Executive Order 14028 (2021) mandates SBOMs for software sold to US federal agencies. Tools like Syft (Anchore) generate SBOMs from container images. Trivy can also generate SBOMs in CycloneDX or SPDX format.

---

**Q23: How do you balance security gate strictness with developer velocity?**

- **What is being tested:** Pragmatic DevSecOps thinking.
- **Strong answer:** The key is making security gates fast, actionable, and with a clear path to resolution. If a gate takes 20 minutes and produces 50 findings with no clear owner, developers learn to ignore it. Good practices: (1) Run lightweight checks (SAST, unit tests) on feature branches for fast feedback — not just at merge, (2) Set thresholds that are achievable — CVSS ≥ 7.0 rather than ≥ 4.0 to avoid alert fatigue, (3) Use `ignore-unfixed` in Trivy to not block on unfixable issues, (4) Keep the full image build pipeline for merged code only — don't run Trivy and ECR push on every feature branch commit, (5) Provide clear error messages with remediation guidance, not just "build failed."

---

## 7. External Secrets Operator

---

## C2

### Question
> "Walk me through how a secret flows from AWS Secrets Manager into a running pod in this setup — step by step."

### What the interviewer is really testing
- Can you trace the full data flow through multiple Kubernetes abstractions?
- Do you understand IRSA at a mechanism level?
- Can you explain CRDs operationally?

---

### Model Answer

The flow has **5 distinct layers**. Most engineers can explain 2-3; knowing all 5 is what gets you the role.

---

### Full step-by-step flow

**Step 1: IRSA authentication — how ESO talks to AWS**

```
ESO controller pod (in kube-system)
    │
    ├── Has ServiceAccount: external-secrets (namespace: kube-system)
    ├── SA has annotation: eks.amazonaws.com/role-arn: arn:aws:iam::516209541629:role/external-secrets-role
    │
    ├── K8s injects projected ServiceAccount token into ESO pod
    │   (via: automountServiceAccountToken)
    │
    └── When ESO calls AWS:
        1. ESO presents SA JWT token to AWS STS
        2. AWS validates JWT signature against cluster's OIDC issuer
           (registered at: oidc.eks.us-east-1.amazonaws.com/id/<cluster-id>)
        3. AWS issues temporary credentials (AccessKeyId, SecretAccessKey, SessionToken)
        4. ESO uses these to call secretsmanager:GetSecretValue
        (no static AWS credentials stored anywhere)
```

**Step 2: ClusterSecretStore defines the connection**

From `k8s/external-secrets/cluster-secret-store.yaml`:
```yaml
spec:
  provider:
    aws:
      service: SecretsManager
      region: us-east-1
      auth:
        jwt:
          serviceAccountRef:
            name: external-secrets
            namespace: kube-system
```
This is the cluster-wide "connector" to AWS Secrets Manager.

**Step 3: ExternalSecret defines what to fetch**

From `k8s/external-secrets/prod-external-secrets.yaml`:
```yaml
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: aws-secrets-manager
    kind: ClusterSecretStore
  target:
    name: db-credentials      # ← name of K8s Secret to create
    creationPolicy: Owner     # ESO owns this Secret's lifecycle
  data:
    - secretKey: DB_USERNAME  # ← key name in K8s Secret
      remoteRef:
        key: /pharma/prod/db-credentials   # ← path in AWS Secrets Manager
        property: username                 # ← field within the JSON secret
```

**Step 4: ESO reconciler creates the Kubernetes Secret**

```
ESO controller loop (runs every refreshInterval):
1. Read ExternalSecret CR
2. Call AWS: GetSecretValue(SecretId: /pharma/prod/db-credentials)
3. AWS returns JSON: {"username": "pharma_user", "password": "s3cr3t"}
4. Extract .username → "pharma_user", .password → "s3cr3t"
5. Create/Update Kubernetes Secret:
   kubectl create secret generic db-credentials \
     --from-literal=DB_USERNAME=pharma_user \
     --from-literal=DB_PASSWORD=s3cr3t \
     -n prod
6. Sets ownerReference on Secret pointing to ExternalSecret
   (if ExternalSecret is deleted, Secret is deleted too — creationPolicy: Owner)
```

**Step 5: Pod consumes the Secret**

From `envs/prod/values-auth-service.yaml`:
```yaml
envFrom:
  - secretRef:
      name: db-credentials   # ← the Secret ESO created
  - secretRef:
      name: jwt-secret
```

Kubernetes injects all keys from `db-credentials` Secret as environment variables into every container in the pod:
```
DB_USERNAME=pharma_user
DB_PASSWORD=s3cr3t
```

---

### Full Architecture Diagram

```
AWS Secrets Manager
  /pharma/prod/db-credentials → {"username": "pharma_user", "password": "s3cr3t"}
  /pharma/prod/jwt-secret     → {"secret": "hs512-key-..."}
         ▲
         │ GetSecretValue (IRSA temp creds)
         │
ESO Controller (kube-system)
  ├── reads ClusterSecretStore
  ├── reads ExternalSecret CRs
  └── writes K8s Secrets (every 1h)
         │
         │ creates/updates
         ▼
K8s Secrets (prod namespace)
  ├── db-credentials: {DB_USERNAME, DB_PASSWORD}
  └── jwt-secret: {JWT_SECRET}
         │
         │ envFrom: secretRef
         ▼
auth-service pod
  └── env: DB_USERNAME, DB_PASSWORD, JWT_SECRET
```

---

## C3

### Question
> "What is IRSA (IAM Roles for Service Accounts) and why is the ClusterSecretStore using JWT auth instead of static credentials?"

### What the interviewer is really testing
- Deep AWS + Kubernetes security knowledge
- Understanding of OIDC federation
- Why static credentials are dangerous and how IRSA eliminates them

---

### Model Answer

**IRSA = IAM Roles for Service Accounts** — a mechanism that allows Kubernetes pods to assume AWS IAM roles without storing any AWS credentials in the cluster.

---

### The problem IRSA solves

**Old approach (static credentials — never do this):**
```yaml
# DANGEROUS — static credentials in a Secret
apiVersion: v1
kind: Secret
metadata:
  name: aws-credentials
data:
  AWS_ACCESS_KEY_ID: AKIAIOSFODNN7EXAMPLE
  AWS_SECRET_ACCESS_KEY: <base64-encoded-key>
```
Problems:
- Key is long-lived — if leaked, attacker has permanent access
- Rotation is manual and painful
- Key has no scope — can't restrict to a specific namespace or pod
- Violates least-privilege — all pods using the SA get the same key

---

### How IRSA works (mechanism)

**Setup (one-time, done via Terraform in this repo):**
```
1. EKS cluster has an OIDC issuer URL:
   https://oidc.eks.us-east-1.amazonaws.com/id/<cluster-id>

2. AWS IAM trusts this OIDC issuer:
   Trust policy on external-secrets-role:
   {
     "Condition": {
       "StringEquals": {
         "oidc.eks.us-east-1.amazonaws.com/id/<id>:sub":
           "system:serviceaccount:kube-system:external-secrets"
       }
     }
   }
   This means: "only the 'external-secrets' SA in 'kube-system' can assume this role"

3. SA is annotated:
   eks.amazonaws.com/role-arn: arn:aws:iam::516209541629:role/external-secrets-role
```

**At runtime (every API call):**
```
ESO pod starts
    │
    ├── K8s injects projected ServiceAccount token (JWT, 1h TTL) into pod
    │   at: /var/run/secrets/eks.amazonaws.com/serviceaccount/token
    │
    └── When ESO calls AWS Secrets Manager:
        1. AWS SDK calls STS AssumeRoleWithWebIdentity(
             RoleArn: arn:aws:iam::516209541629:role/external-secrets-role,
             WebIdentityToken: <SA JWT>
           )
        2. AWS STS validates JWT signature against OIDC issuer public keys
        3. Checks trust policy conditions (namespace + SA name match)
        4. Returns temporary credentials (15min to 1h TTL)
        5. ESO uses temp creds for Secrets Manager API call
```

---

### Why this is better than static credentials

| Property | Static Credentials | IRSA |
|----------|-------------------|------|
| Credential lifetime | Permanent (until manually rotated) | 15 minutes to 1 hour |
| Scope | Attached to key, not to identity | Bound to specific namespace + SA name |
| Storage | In a K8s Secret (exploitable) | Never stored — generated on demand |
| Rotation | Manual, high-risk | Automatic — tokens auto-rotate |
| Audit trail | AWS CloudTrail shows access key ID | CloudTrail shows role + cluster + SA identity |
| Blast radius if leaked | Full AWS account access | Only the ESO role permissions, for max 1h |

---

### From the ClusterSecretStore in this repo

```yaml
auth:
  jwt:
    serviceAccountRef:
      name: external-secrets
      namespace: kube-system
```

This tells ESO: "use the JWT from the `external-secrets` ServiceAccount in `kube-system` to authenticate." This is the IRSA mechanism — the JWT is the web identity token in the `AssumeRoleWithWebIdentity` call.

---

## 8. TLS/SSL Certificate Management

---

## C4

### Question
> "The pharma portal's TLS is terminated by NGINX Ingress inside the cluster, behind a pass-through NLB. Where exactly is the certificate stored, how is it provisioned and renewed, and why can't you just use AWS ACM here?"

### What the interviewer is really testing
- Whether you understand TLS termination as an architectural decision, not just "some load balancer handles it"
- Whether you know ACM's actual constraint (non-exportable private key), not just "ACM = AWS-managed certs"
- Awareness that certificate management has more than one legitimate real-world pattern, not just one "correct" answer

---

### Model Answer

**Storage:** the certificate and private key live in a Kubernetes `Secret` of type `kubernetes.io/tls` (`tls.crt` + `tls.key`), in the same namespace as the Ingress. The Ingress resource references it via `spec.tls[].secretName`; the NGINX Ingress Controller watches that Secret through the Kubernetes API and loads it into its running nginx TLS context, reloading automatically when the Secret changes — no file mount, no manual restart required.

**Why ACM is off the table for this specific architecture.** ACM never lets you export a certificate's private key — by design, it only works when AWS itself terminates the TLS handshake (an ALB TLS listener, an NLB in TLS-listener mode, CloudFront, API Gateway). Here the NLB is a pure **Layer 4 pass-through** — it forwards raw TCP to node ports — and NGINX inside the cluster does the actual termination. Since nginx needs the real private key material and ACM will never hand that over, ACM simply cannot be the source of this certificate — regardless of whether the domain is public or internal. ACM only becomes usable if the architecture changes to put an ALB or CloudFront in front instead.

**Two legitimate provisioning patterns for the in-cluster-termination architecture:**

1. **cert-manager + Let's Encrypt (ACME).** A `Certificate` + `ClusterIssuer` resource; cert-manager runs the ACME HTTP-01/DNS-01 challenge, writes the issued cert straight into the Secret, and silently re-issues it ~30 days before expiry. Fully automated and free, but only a DV (domain-validated) certificate — no organization-identity verification.

2. **A purchased CA certificate (DigiCert/Sectigo/etc.) synced via External Secrets Operator.** Many regulated pharma/finance orgs require an OV or EV certificate with a documented issuance trail for compliance, which Let's Encrypt doesn't provide. The key pair is generated once, the cert is issued out-of-band by the CA, stored in AWS Secrets Manager, and synced into the same `kubernetes.io/tls`-shaped Secret via ESO — reusing the exact pipeline this project already uses for `jwt-secret` and `db-credentials`.

**The renewal difference is the part worth emphasizing.** Pattern 1 renews itself with zero human involvement. Pattern 2 does not — a purchased cert is typically 1–2 year validity, renewal requires a human-initiated request to the CA, and nothing but a monitoring alert on the certificate's `notAfter` date stands between "fine" and a silent expiry months later — there's no cert-manager `Certificate` object to show `READY=False`, so you have to alert directly off the cert's own expiry timestamp.

**Security controls that apply either way:** RBAC should restrict `get`/`describe` on TLS secrets (they contain a private key), and EKS should have KMS envelope encryption enabled on etcd so the Secret is encrypted at rest, not just base64-encoded.
