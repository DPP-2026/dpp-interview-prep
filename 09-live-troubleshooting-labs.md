# Live Troubleshooting Labs — Break It, Investigate It, Fix It

> These are hands-on labs, not just reading material. Each one gives you a real incident description exactly as an interviewer would phrase it, the commands to **reproduce the failure yourself**, the live diagnostic path, and the fix — so you can practice the actual muscle memory instead of memorizing a story.
>
> **Only ever run the "Break it" steps against a personal or throwaway sandbox you fully control** (a scratch EKS/kind cluster, an isolated AWS sandbox account). Never run them against shared dev/qa, and never against anything resembling production. Several steps modify IAM policies, security groups, and Route53 records — treat them with the same care you would in a real account.
>
> This file deliberately goes **beyond Kubernetes**. Interviewers test the layers around the cluster just as often as the cluster itself — DNS, the load balancer, and the CI/CD system. For pure Kubernetes pod/scheduling incidents (`CrashLoopBackOff`, `Pending`, `ImagePullBackOff`, `OOMKilled`, node `NotReady`, ArgoCD drift), see the **"Incident Response Scenarios"** section of [`02-eks-kubernetes.md`](./02-eks-kubernetes.md). TLS/SSL certificate storage and management (cert-manager vs. purchased CA certs, why ACM doesn't fit an NLB pass-through architecture) is architectural rather than something you'd reproduce live — see `05-security-devsecops.md` C4 for that instead.
>
> **Why the questions are grouped, not split one-cause-per-question.** Real interviews at Google, Amazon, Microsoft, Meta, Netflix, Uber, and large regulated shops like JPMorgan and Goldman Sachs almost never ask "what causes X" for a single narrow cause — the standard senior SRE/Platform prompt is "here's a symptom, walk me through *every* plausible cause, in order of likelihood, and how you'd isolate each one." Answering with only one cause when three exist reads as inexperience, even if the one cause you name is correct. Each question below is grouped that way on purpose, with a decision tree at the end — that's the actual shape of the question you'll be asked, not an artificial simplification.

---

## Table of Contents

1. [DNS](#1-dns) — Q1
2. [Load Balancer & Ingress](#2-load-balancer--ingress) — Q2–Q3
3. [Database](#3-database) — Q4
4. [Secrets Management](#4-secrets-management) — Q5
5. [CI/CD Outside Kubernetes](#5-cicd-outside-kubernetes) — Q6–Q7

---

## 1. DNS

---

**Q1. Two DNS complaints land at once: a public API domain resolves to the wrong, decommissioned load balancer, and separately, pods intermittently fail to resolve an internal RDS hostname. Walk me through investigating DNS end-to-end — from the public record down to in-cluster resolution — and fixing each layer.**

**Asked in this style at:** Amazon and Google in particular love "trace the full resolution path" framing for SRE/Infra roles — it tests whether you treat DNS as a stack of independent layers (authoritative record → resolver cache → in-cluster resolver) instead of one opaque black box you restart when it's broken.

**What is being tested:** Whether you can tell apart an external record problem (Route53/IaC drift) from an internal one (CoreDNS capacity) using the right tool for each layer, rather than randomly restarting things.

---

### Cause 1: External record drift (Route53 record doesn't match Terraform state)

**Break it (reproduce live):**

```bash
aws route53 list-hosted-zones-by-name --dns-name zen-pharma.com.

# Point the record at a decommissioned/nonexistent NLB, simulating console drift outside Terraform
aws route53 change-resource-record-sets --hosted-zone-id Z0123456789ABC \
  --change-batch '{
    "Changes": [{
      "Action": "UPSERT",
      "ResourceRecordSet": {
        "Name": "api.zen-pharma.com",
        "Type": "A",
        "AliasTarget": {
          "HostedZoneId": "Z35SXDOTRQ7X7K",
          "DNSName": "old-decommissioned-nlb-1234567890.elb.us-east-1.amazonaws.com",
          "EvaluateTargetHealth": true
        }
      }
    }]
  }'
```

**Investigate:**

```bash
dig api.zen-pharma.com +short
dig api.zen-pharma.com @8.8.8.8 +short         # bypass any local resolver cache, ask a public resolver directly
dig api.zen-pharma.com +trace                  # walk the delegation chain, confirm which nameserver is answering

# Compare the live record against Terraform's declared state
cd zen-infra && terraform plan -target=aws_route53_record.api
# Plan: 1 to change  → confirms drift: something changed this outside the PR-reviewed flow

aws elbv2 describe-load-balancers --names old-decommissioned-nlb 2>&1
# An error occurred (LoadBalancerNotFound) — confirms the target doesn't even exist anymore
```

**Root cause:** a manual console change (or a `terraform apply` run from a stale branch whose module output still referenced the old NLB) overwrote the Route53 alias record outside the normal PR-reviewed flow.

**Fix:**

```bash
terraform apply -target=aws_route53_record.api
dig api.zen-pharma.com +short
```

---

### Cause 2: Internal resolver capacity (CoreDNS overloaded)

**Break it (reproduce live):**

```bash
kubectl scale deployment coredns -n kube-system --replicas=1
kubectl set resources deployment coredns -n kube-system -c coredns --limits=cpu=50m,memory=30Mi

kubectl create job dns-stress --image=busybox --dry-run=client -o yaml \
  -- /bin/sh -c "for i in $(seq 1 5000); do nslookup drug-catalog-db.xxxxx.us-east-1.rds.amazonaws.com; done" \
  | kubectl apply -f -
```

**Investigate:**

```bash
kubectl get pods -n kube-system -l k8s-app=kube-dns
kubectl top pod -n kube-system -l k8s-app=kube-dns          # check for CPU throttling / OOM proximity
kubectl logs -n kube-system -l k8s-app=kube-dns --since=5m | grep -i "error\|timeout"

kubectl exec -n prod deploy/manufacturing-service -- \
  sh -c 'for i in $(seq 1 20); do nslookup drug-catalog-db.xxxxx.us-east-1.rds.amazonaws.com; done' \
  | grep -c "can't resolve\|NXDOMAIN"
```

**Root cause:** CoreDNS scaled to a single replica with undersized CPU/memory limits gets throttled under load; UDP queries landing during a throttled window get dropped, while the rest succeed — hence *intermittent*, not total, failure.

**Fix:**

```bash
kubectl scale deployment coredns -n kube-system --replicas=2
kubectl set resources deployment coredns -n kube-system -c coredns \
  --limits=cpu=200m,memory=170Mi --requests=cpu=100m,memory=70Mi
```

Longer term: install the `cluster-proportional-autoscaler` for CoreDNS, and enable app-level DNS result caching (JVM `networkaddress.cache.ttl`) so not every request performs a fresh lookup.

---

### Decision tree

```
DNS-looking symptom
       │
       ├── One specific external domain resolves to a wrong/stale target
       │       → dig +trace / dig @8.8.8.8, compare against `terraform plan`
       │       → Cause 1: Route53 record drifted outside IaC
       │
       └── In-cluster lookups intermittently fail (some succeed, some don't)
               → kubectl get pods/top pod -n kube-system -l k8s-app=kube-dns
               → Cause 2: CoreDNS under-provisioned / no HA / query-volume throttling
```

<details>
<summary>📘 Teaching note</summary>

**Why `dig @8.8.8.8` matters for Cause 1.** The record is an **Alias**, so Route53 itself re-evaluates it on every query — no caching at that layer. But resolvers *downstream* (OS, corporate DNS, browser) still cache their last answer for their own TTL. Querying a public resolver directly sidesteps a possibly-stale local cache, so you don't mistake "my fix didn't work" for "my fix worked but my laptop's cache hasn't expired yet."

</details>

---

## 2. Load Balancer & Ingress

---

**Q2. The load balancer reports target(s) unhealthy. Walk me through every plausible cause, from most to least likely, and how you'd isolate each one.**

**Asked in this style at:** Google, Meta, and Uber SRE interviews standardize on exactly this "give me the full decision tree, not just the one cause you personally hit" phrasing — they're checking whether your mental model spans the app, Kubernetes, and network layers, not just whichever single bug you remember from a past incident.

**What is being tested:** Systematic, layer-by-layer elimination — application, then Kubernetes, then network — instead of guessing based on the last outage you happened to see.

---

### Cause 1 (most common): application-level — readiness probe misconfigured

This is the single most common real-world cause and is covered in full depth, with the exact incident narrative, in [`07-behavioral-realtime.md`](./07-behavioral-realtime.md) Q5 and G2 — a Helm values change pointed the readiness probe at a path that doesn't exist (`/health/ready` instead of `/actuator/health/readiness`), pods stayed `Running` but were marked `NotReady`, and the load balancer correctly stopped sending them traffic. Worth re-reading here because it's the first thing to rule out:

```bash
kubectl get pods -n prod -o wide          # Running but check READY column, not just STATUS
kubectl describe pod <pod> -n prod | grep -A5 Readiness
```

### Cause 2: the Ingress controller itself is down — total outage, not partial

**Break it (reproduce live):**

```bash
# An invalid boolean value breaks NGINX's config template validation on the next reload
kubectl patch configmap ingress-nginx-controller -n ingress-nginx --type merge -p \
  '{"data":{"use-forwarded-headers":"not-a-boolean"}}'
kubectl rollout restart deployment ingress-nginx-controller -n ingress-nginx
```

**Investigate:**

```bash
kubectl get pods -n ingress-nginx
# ingress-nginx-controller-xxxx   0/1   CrashLoopBackOff

kubectl logs -n ingress-nginx <pod> --previous | tail -30
# nginx: [emerg] invalid value "not-a-boolean" in .../nginx.conf

kubectl get configmap ingress-nginx-controller -n ingress-nginx -o yaml
# diff this against the last known-good version tracked in zen-gitops

aws elbv2 describe-target-health --target-group-arn <nlb-tg-arn>
# All targets unhealthy — because no controller pod behind them is Ready
```

**Root cause:** an invalid ConfigMap value broke NGINX's config template validation. Every controller replica that restarted picked up the bad config and failed to start `nginx`, taking down routing for **every** Ingress resource in the cluster simultaneously — the tell here is that *all* services are down at once, not just one.

**Fix:**

```bash
kubectl patch configmap ingress-nginx-controller -n ingress-nginx --type merge -p \
  '{"data":{"use-forwarded-headers":"true"}}'
kubectl rollout restart deployment ingress-nginx-controller -n ingress-nginx
kubectl rollout status deployment ingress-nginx-controller -n ingress-nginx
```

### Cause 3 (hardest to spot): security group blocking the NodePort range

**Break it (reproduce live):**

```bash
aws ec2 revoke-security-group-ingress \
  --group-id sg-0123456789nodesg \
  --protocol tcp --port 30000-32767 \
  --cidr 10.0.0.0/16
```

**Investigate:**

```bash
kubectl get pods -n prod -o wide                       # Running, Ready — everything K8s-visible is healthy
kubectl get endpoints drug-catalog-service -n prod      # Endpoints populated correctly

aws elbv2 describe-target-health --target-group-arn <tg-arn>
# TargetHealth: unhealthy, Reason: Target.Timeout

# From a bastion/EC2 instance in the same VPC — test the network path directly
nc -zv <node-private-ip> <node-port>
# Connection timed out   ← proves this is a network-path problem, not app or Kubernetes

aws ec2 describe-security-groups --group-ids sg-0123456789nodesg \
  --query "SecurityGroups[0].IpPermissions"
# Missing the NodePort-range ingress rule that should be there
```

**Root cause:** a Terraform change to the node security group (e.g. during a compliance tightening pass) removed the rule allowing the NLB's health checks — and real traffic — to reach the node's NodePort range. Every Kubernetes-level object is completely healthy, which is exactly why this one is hard: nothing `kubectl` can see is wrong.

**Fix:**

```bash
git revert <bad-sg-commit> && terraform apply
```

---

### Decision tree

```
LB reports target(s) unhealthy
       │
       ├── kubectl get pods → Running but NOT Ready
       │       → Cause 1: readiness probe misconfigured (see 07-behavioral-realtime.md Q5/G2)
       │
       ├── kubectl get pods → not even Running (CrashLoopBackOff)
       │       → Cause 2: ingress controller itself broken — check its own logs/ConfigMap
       │
       └── kubectl get pods → Running AND Ready, Endpoints populated, LB still times out
               → Cause 3: network path blocked — check security groups, NACLs, route tables
               → test with `nc`/`telnet` from an instance in the same VPC, not with kubectl
```

<details>
<summary>📘 Teaching note</summary>

**The general rule to state out loud in an interview:** narrow top-down — app health, then K8s object health, then network path — and only escalate to the next layer once the current one is *proven* healthy, not assumed. Cause 3 is the one that separates senior candidates: most people stop checking once `kubectl` says everything is fine, without remembering that `kubectl` has no visibility into security groups or NACLs at all.

</details>

---

**Q3. `/api/v2/inventory` returns 404 through the Ingress, but every other endpoint on `drug-catalog-service` works fine. Reproduce it, investigate, fix.**

**Asked in this style at:** Microsoft and Amazon — a very common "is this actually the same kind of 404 you're used to" trap question, because most candidates jump straight to application logs.

**What is being tested:** Whether you can tell an Ingress routing bug apart from an application bug — both present as "404," but they require completely different fixes.

**Break it (reproduce live):**

```bash
kubectl get ingress drug-catalog-service -n prod -o yaml > /tmp/ingress-backup.yaml
kubectl patch ingress drug-catalog-service -n prod --type=json -p '[
  {"op":"replace","path":"/spec/rules/0/http/paths/1/path","value":"/api/v2/inventoryyy"}
]'
```

**Investigate:**

```bash
curl -v https://api.zen-pharma.com/api/v2/inventory
# HTTP/1.1 404 Not Found
# < Server: nginx     ← the 404 comes from the Ingress, not the Spring Boot app

kubectl describe ingress drug-catalog-service -n prod
# Rules:
#   Path                     Backends
#   ----                     --------
#   /api/v2/inventoryyy      drug-catalog-service:8080     ← doesn't match what clients actually call

kubectl exec -n ingress-nginx deploy/ingress-nginx-controller -- \
  cat /etc/nginx/nginx.conf | grep -B2 -A5 "inventory"
# confirms there is no location block matching the real, requested path
```

**Root cause:** the Ingress `path` field was edited (or a Helm values merge went wrong), so the declared route no longer matches what clients request. NGINX correctly returns 404 for any request with no matching `location` block — this is nginx's 404, not the application's, which is the key diagnostic signal (`Server: nginx` in the response header, plus every other path to the same Service working normally).

**Fix:** `kubectl apply -f /tmp/ingress-backup.yaml` (in real life: `git revert` the offending zen-gitops PR, then `argocd app sync`).

**Verify:** curl returns 200.

---

## 3. Database

---

**Q4. The database layer is misbehaving under normal operation — walk me through diagnosing connection issues, both under steady load and right after a failover.**

**Asked in this style at:** JPMorgan, Goldman Sachs, and Amazon lean heavily on DB connection-pool behavior for backend/platform roles — it's one of the most reliable "have you actually operated this in production" filters, since it's very hard to fake without hands-on experience.

**What is being tested:** Whether you check both structural causes (pool size × replica count) and time-of-event causes (failover) rather than treating "database problem" as one monolithic category.

---

### Cause 1: connection pool exhaustion under steady load

**Break it (reproduce live, dev/sandbox RDS only):**

```bash
for i in $(seq 1 25); do
  psql "host=drug-catalog-db-dev.xxxxx.us-east-1.rds.amazonaws.com dbname=catalog user=app" \
    -c "SELECT pg_sleep(600);" &
done
```

**Investigate:**

```bash
kubectl logs -n dev deploy/manufacturing-service --since=5m | grep -i "too many\|connection"
# FATAL: sorry, too many clients already

psql "$RDS_ADMIN_URI" -c "SELECT count(*), state FROM pg_stat_activity GROUP BY state;"
#  count | state
#  ------+-------
#     20 | idle

psql "$RDS_ADMIN_URI" -c "SHOW max_connections;"
kubectl get deployment manufacturing-service -n dev -o jsonpath='{.spec.replicas}'
kubectl exec -n dev deploy/manufacturing-service -- env | grep -i hikari
# HIKARI_MAXIMUM_POOL_SIZE=10, replicas=3 → up to 30 possible connections against a lower connection ceiling
```

**Root cause:** two independent causes to check every time — (1) `replica_count × pool_size` for a single service can already exceed what the instance allows, and gets worse the more services share one RDS instance; (2) idle or abandoned sessions (a transaction never closed, a debugging `psql` session left open) hold connection slots indefinitely regardless of pool size.

**Fix:**

```bash
# Immediate relief — reclaim connections held by long-idle sessions
psql "$RDS_ADMIN_URI" -c "
  SELECT pg_terminate_backend(pid) FROM pg_stat_activity
  WHERE state = 'idle' AND state_change < now() - interval '10 minutes';"

# Structural fix — right-size the pool per replica, or add PgBouncer in front of RDS
kubectl set env deployment/manufacturing-service -n dev HIKARI_MAXIMUM_POOL_SIZE=5
```

### Cause 2: failover — the database recovers, the application doesn't

**Break it (reproduce live, dev/sandbox RDS only):**

```bash
aws rds reboot-db-instance --db-instance-identifier drug-catalog-db-dev --force-failover
```

**Investigate:**

```bash
aws rds describe-events --source-identifier drug-catalog-db-dev --source-type db-instance --duration 20

kubectl logs -n dev deploy/drug-catalog-service --since=2m -f
# HikariPool-1 - Connection is not available, request timed out

psql "$RDS_ADMIN_URI" -c "SELECT 1;"     # DB itself is already healthy again — confirms the gap is app-side

kubectl exec -n dev deploy/drug-catalog-service -- env | grep -i "VALIDATION\|MAX_LIFETIME"
# HIKARI_VALIDATION_TIMEOUT=250000   (250s) ← pool keeps handing out stale connections for minutes
```

**Root cause:** HikariCP has no awareness that a failover changed which physical instance the RDS endpoint DNS name points to. Existing pooled connections to the old primary aren't automatically dropped — they're only evicted once Hikari's own `validationTimeout`/`maxLifetime` checks catch them as dead. A window set too high means the pool keeps handing out connections it believes are healthy for minutes after the database itself has already recovered.

**Fix:**

```bash
kubectl set env deployment/drug-catalog-service -n dev HIKARI_VALIDATION_TIMEOUT=5000 HIKARI_MAX_LIFETIME=300000
kubectl rollout restart deployment drug-catalog-service -n dev
```

Add a Grafana panel on `hikaricp_connections_active` / `hikaricp_connections_timeout` so this gap is visible immediately next time instead of inferred from logs after the fact.

---

### Decision tree

```
DB-layer symptom
       │
       ├── "too many clients" / connections refused, under normal load
       │       → psql: SELECT count(*), state FROM pg_stat_activity GROUP BY state;
       │       → Cause 1: pool size × replicas > max_connections, or idle-session leak
       │
       └── Errors start right after a maintenance window / AZ event, DB itself responds fine
               → psql "SELECT 1;" succeeds, but app still errors
               → Cause 2: connection pool hasn't evicted stale connections to the old primary
```

---

## 4. Secrets Management

---

**Q5. `kubectl get externalsecret -A` shows `jwt-secret` in `prod` as `SecretSyncedError`. The Kubernetes Secret still exists, but pods reading it are getting stale/empty values. Reproduce it, investigate, fix.**

**Asked in this style at:** Google and Microsoft — IRSA/Workload Identity exact-string-matching bugs are a favorite "small typo, big blast radius" question because they can't be answered from documentation alone; you have to have actually hit one.

**What is being tested:** Whether you understand that ESO failing to sync does **not** delete the existing Secret — which is exactly why this bug is confusing: it looks like "the secret went empty," not "the secret disappeared."

**Break it (reproduce live):**

```bash
aws iam get-role-policy --role-name external-secrets-irsa-role --policy-name eso-secretsmanager-read \
  > /tmp/eso-policy-backup.json

aws iam put-role-policy --role-name external-secrets-irsa-role \
  --policy-name eso-secretsmanager-read \
  --policy-document '{
    "Version": "2012-10-17",
    "Statement": [{"Effect":"Deny","Action":"secretsmanager:GetSecretValue","Resource":"*"}]
  }'
```

**Investigate:**

```bash
kubectl get externalsecret -A
# NAMESPACE   NAME         STORE          STATUS
# prod        jwt-secret   aws-secrets    SecretSyncedError

kubectl describe externalsecret jwt-secret -n prod
# Message: AccessDeniedException: User: arn:aws:sts::516209541629:assumed-role/external-secrets-irsa-role
#          is not authorized to perform: secretsmanager:GetSecretValue

kubectl logs -n external-secrets deploy/external-secrets --since=10m | grep jwt-secret

kubectl run debug --rm -it --image=amazon/aws-cli -n external-secrets \
  --overrides='{"spec":{"serviceAccountName":"external-secrets-sa"}}' -- sts get-caller-identity
```

**Root cause:** an IAM policy change (accidental, or an overly broad `terraform apply` from an unrelated module) added an explicit `Deny` for `secretsmanager:GetSecretValue` on the ESO IRSA role. ESO keeps retrying on its refresh interval and keeps failing — but it never deletes the Kubernetes Secret it manages, so pods that already have it mounted keep running on stale data until they restart, at which point they get an empty value instead of a "missing secret" error.

**Fix:**

```bash
aws iam put-role-policy --role-name external-secrets-irsa-role \
  --policy-name eso-secretsmanager-read --policy-document file:///tmp/eso-policy-backup.json

kubectl annotate externalsecret jwt-secret -n prod force-sync=$(date +%s) --overwrite
kubectl describe externalsecret jwt-secret -n prod    # Status flips back to SecretSynced
kubectl rollout restart deployment auth-service -n prod
```

**Verify:** `kubectl get secret jwt-secret -n prod -o jsonpath='{.data.JWT_SECRET}' | base64 -d` returns a non-empty value; app logs clean.

---

## 5. CI/CD Outside Kubernetes

---

**Q6. The deploy job in GitHub Actions starts failing with `Error: Could not assume role with OIDC: Not authorized to perform sts:AssumeRoleWithWebIdentity`, right after a Terraform change to the CI IAM role. Reproduce it, investigate, fix.**

**Asked in this style at:** Netflix, Uber, and Meta — OIDC federation bugs are a common "prove you understand the trust boundary, not just the YAML" question for platform/infra roles building internal CI systems.

**What is being tested:** Whether you know OIDC federation does exact string matching on the JWT's `sub` claim — the same failure class as the IRSA story in `07-behavioral-realtime.md` Q6 and Q5 above, one layer up the stack.

**Break it (reproduce live):**

```hcl
# in the aws_iam_role trust policy — introduce the bug on purpose
condition {
  test     = "StringEquals"
  variable = "token.actions.githubusercontent.com:sub"
  values   = ["repo:ravdy/zen-pharma-backend:ref:refs/heads/release"]  # was refs/heads/main
}
```
`terraform apply`, then push to `main` to trigger the deploy workflow.

**Investigate:**

```bash
gh run list --workflow=deploy.yml --limit 5
gh run view <run-id> --log-failed
# Error: Could not assume role with OIDC: Not authorized to perform sts:AssumeRoleWithWebIdentity

aws iam get-role --role-name gha-deploy-role --query 'Role.AssumeRolePolicyDocument'
# "sub": "repo:ravdy/zen-pharma-backend:ref:refs/heads/release"   ← doesn't match "refs/heads/main"

grep -A3 "permissions:" .github/workflows/deploy.yml
# id-token: write   ← present and correct, so this isn't the "missing permission" flavor of this bug
```

**Root cause:** the trust policy was updated to trust a different branch ref than the one actually pushing. Every workflow run on `main` presents a token whose `sub` claim the role's trust condition simply doesn't match.

**Fix:**

```hcl
condition {
  test     = "StringEquals"
  variable = "token.actions.githubusercontent.com:sub"
  values   = ["repo:ravdy/zen-pharma-backend:ref:refs/heads/main"]
}
```
`terraform apply`, then `gh run rerun <run-id>`.

**Verify:** `gh run view <new-run-id>` shows the AssumeRole step succeeding.

---

**Q7. A CI/CD job is stuck — not failing, not erroring, just sitting there indefinitely. Walk me through investigating this on GitHub Actions, and how the same investigation maps if the interviewer's company runs Jenkins instead.**

**Asked in this style at:** large regulated shops still running Jenkins (banks, insurers, older tech giants mid-migration) ask the Jenkins half directly; Big Tech infra teams (Google, Amazon, Meta) ask the "map it to a tool you don't use" half specifically to test conceptual vs. memorized knowledge.

**What is being tested:** Whether you check the *gate/queue configuration* — not the logs — once you recognize "stuck" as a different failure mode from "failing," and whether you can generalize that instinct across CI systems.

---

### Cause 1: GitHub Actions — stuck at an environment approval gate

**Break it (reproduce live):**

```bash
gh api -X PUT /repos/ravdy/zen-pharma-backend/environments/prod \
  -f "reviewers[][type]=Team" -F "reviewers[][id]=99999999"   # a team ID that no longer exists
```

**Investigate:**

```bash
gh run list --limit 5
gh run view <run-id>
# Job "deploy-prod" — Waiting for approval — no error message anywhere

gh api /repos/ravdy/zen-pharma-backend/environments/prod
# "protection_rules": [{"type":"required_reviewers","reviewers":[{"type":"Team","id":99999999}]}]

gh api /orgs/ravdy/teams -q '.[].id'   # confirm 99999999 doesn't correspond to any active team
```

**Root cause:** the job is parked at an `environment: prod` approval gate waiting for a reviewer team that was deleted (or a typo'd ID from a change to environment protection rules). No one can ever approve it, so it sits until the workflow's timeout eventually kills it.

**Fix:**

```bash
gh api -X PUT /repos/ravdy/zen-pharma-backend/environments/prod \
  -f "reviewers[][type]=Team" -F "reviewers[][id]=<real-release-managers-team-id>"
gh run rerun <run-id>
```

### Cause 2: Jenkins — stuck in the build queue

**Concept mapping — lead with this if the interviewer's shop runs Jenkins:**

| GitHub Actions concept | Jenkins equivalent |
|---|---|
| Runner (GitHub-hosted or self-hosted) | Agent / Node |
| Job concurrency limits | Executor count per agent |
| `.github/workflows/*.yml` | Jenkinsfile |
| `environment:` approval gate | Input step / Lockable Resources |
| OIDC role trust policy | Credentials binding / cloud IAM plugin |

**Break it (if you have a real Jenkins sandbox):**

```bash
sudo systemctl stop jenkins-agent
# or keep every executor busy:
for i in 1 2 3 4; do curl -X POST "$JENKINS_URL/job/busy-job/build" --user "$USER:$TOKEN"; done
```

**Investigate:**

```bash
curl -s "$JENKINS_URL/computer/api/json?pretty=true" --user "$USER:$TOKEN" \
  | jq '.computer[] | {displayName, offline, numExecutors}'

curl -s "$JENKINS_URL/queue/api/json?pretty=true" --user "$USER:$TOKEN" \
  | jq '.items[] | {why, stuck, blocked}'
# "why": "Waiting for next available executor on ‘agent-2’"

curl -s "$JENKINS_URL/computer/agent-2/api/json" --user "$USER:$TOKEN" | jq '.offlineCauseReason'
```

**Root cause categories, in order of likelihood:** (1) agent disconnected — network/firewall or expired JNLP secret; (2) all executors on matching agents busy — a hung job with no timeout never released its executor; (3) label mismatch — the job's `agent { label 'x' }` doesn't match any connected node; (4) disk pressure — Jenkins won't schedule below its configured free-disk-space threshold.

**Fix:** reconnect the agent, abort the hung job holding the executor, correct the node label, or free disk space (Workspace Cleanup plugin / prune old builds).

---

### Decision tree

```
CI job status = stuck, no error in logs
       │
       ├── GitHub Actions: `gh run view` says "Waiting for approval"
       │       → Cause 1: check environment protection_rules — reviewer team may not exist
       │
       └── Jenkins: job sits in queue, never picks up an agent
               → Cause 2: check queue API "why" field — offline agent, full executors, label
                 mismatch, or disk pressure, in that order
```

<details>
<summary>📘 Teaching note</summary>

Worth stating explicitly in an interview: "stuck" and "failing" are different failure modes requiring different first moves. A failing job means read the logs. A stuck job means the logs will be empty — go straight to the scheduler/gate configuration instead. The strongest answer to the tool-bridging half of this question is naming the mapping table out loud: "the mental model transfers 1:1 — runner/agent health, concurrency limits, and approval gates are the three places I'd check on any CI system."

</details>
