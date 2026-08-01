# Live Troubleshooting Labs — Break It, Investigate It, Fix It

> These are hands-on labs, not just reading material. Each one gives you a real incident description exactly as an interviewer would phrase it, the commands to **reproduce the failure yourself**, the live diagnostic path, and the fix — so you can practice the actual muscle memory instead of memorizing a story.
>
> **Only ever run the "Break it" steps against a personal or throwaway sandbox you fully control** (a scratch EKS/kind cluster, an isolated AWS sandbox account). Never run them against shared dev/qa, and never against anything resembling production. Several steps modify IAM policies, security groups, and Route53 records — treat them with the same care you would in a real account.
>
> This file deliberately goes **beyond Kubernetes**. Interviewers test the layers around the cluster just as often as the cluster itself — DNS, TLS/certificates, the load balancer, and the CI/CD system. For pure Kubernetes pod/scheduling incidents (`CrashLoopBackOff`, `Pending`, `ImagePullBackOff`, `OOMKilled`, node `NotReady`, ArgoCD drift), see the **"Incident Response Scenarios"** section of [`02-eks-kubernetes.md`](./02-eks-kubernetes.md) — this file picks up everywhere else.

---

## Table of Contents

1. [DNS & TLS](#1-dns--tls) — Q1–Q3
2. [Load Balancer & Ingress](#2-load-balancer--ingress) — Q4–Q6
3. [Database](#3-database) — Q7–Q8
4. [Secrets Management](#4-secrets-management) — Q9
5. [CI/CD Outside Kubernetes](#5-cicd-outside-kubernetes) — Q10–Q12

---

## 1. DNS & TLS

---

**Q1. The pharma portal (`https://portal.zen-pharma.internal`) suddenly shows a browser TLS warning — "Your connection is not private, NET::ERR_CERT_DATE_INVALID." Reproduce this live, investigate it, and fix it.**

**What is being tested:** Whether you understand cert-manager's ACME renewal flow well enough to find *why* renewal failed, instead of just manually swapping in a new cert and never learning the root cause.

**Break it (reproduce live):**

Fastest way to get the exact symptom without waiting on ACME — install an already-expired self-signed cert directly into the secret the Ingress serves:

```bash
faketime '2024-01-01 00:00:00' openssl req -x509 -newkey rsa:2048 -nodes \
  -keyout tls.key -out tls.crt -days 30 \
  -subj "/CN=portal.zen-pharma.internal"

kubectl create secret tls portal-tls -n prod \
  --cert=tls.crt --key=tls.key --dry-run=client -o yaml | kubectl apply -f -
```

To reproduce the *realistic* root cause (a broken ACME renewal, not just a manually swapped cert), shadow cert-manager's HTTP-01 challenge path with a higher-priority Ingress rule, then force an early renewal:

```bash
kubectl annotate certificate portal-tls -n prod cert-manager.io/issue-temporary-certificate="true" --overwrite
kubectl delete secret portal-tls -n prod   # forces cert-manager to reissue
```

**Investigate:**

```bash
curl -vI https://portal.zen-pharma.internal 2>&1 | grep -i "certificate\|expire"
# curl: (60) SSL certificate problem: certificate has expired

echo | openssl s_client -connect portal.zen-pharma.internal:443 -servername portal.zen-pharma.internal 2>/dev/null \
  | openssl x509 -noout -dates
# notBefore=Jan  1 00:00:00 2024 GMT
# notAfter=Jan 31 00:00:00 2024 GMT      ← expired well over a year ago

kubectl get certificate -n prod
# NAME         READY   SECRET       AGE
# portal-tls   False   portal-tls   400d

kubectl describe certificate portal-tls -n prod
kubectl get order,challenge -n prod
kubectl describe challenge <challenge-name> -n prod
# Reason: Waiting for HTTP-01 challenge propagation: failed to perform self check GET request... 404

kubectl logs -n cert-manager deploy/cert-manager --since=10m | grep -i portal-tls
```

**Root cause:** cert-manager renews `renewBefore` days ahead of expiry. Renewal requires Let's Encrypt to reach `http://portal.zen-pharma.internal/.well-known/acme-challenge/<token>` and hit cert-manager's ephemeral solver pod. A conflicting Ingress rule (or an NGINX `configuration-snippet` someone added) intercepted that path first, the challenge 404'd, the renewal `Order` never completed, and the existing certificate kept aging past `notAfter` with no visible error until a user hit the browser warning.

**Fix:**

```bash
kubectl delete ingress acme-challenge-override -n prod   # or: git revert the offending zen-gitops PR + argocd app sync
kubectl delete challenge --all -n prod
kubectl delete certificaterequest --all -n prod
kubectl delete secret portal-tls -n prod   # cert-manager reissues cleanly now the path is clear
kubectl describe certificate portal-tls -n prod   # watch Ready flip to True
```

**Verify & clean up:**

```bash
echo | openssl s_client -connect portal.zen-pharma.internal:443 -servername portal.zen-pharma.internal 2>/dev/null | openssl x509 -noout -dates
curl -vI https://portal.zen-pharma.internal
```

<details>
<summary>📘 Teaching note</summary>

**Why this fails silently for weeks before anyone notices.** `renewBefore` (commonly 30 days) means you get a full month of "everything looks fine" before a broken ACME solver path becomes user-facing. The only proactive signal is `kubectl get certificate -A` showing `READY=False`, or a Prometheus alert on `certmanager_certificate_expiration_timestamp_seconds`. Waiting for the browser warning means you already missed a month of warning signs — this is the argument for alerting on certificate expiry directly rather than trusting "someone will notice."

</details>

---

**Q2. `api.zen-pharma.com` used to resolve to the prod load balancer; now it resolves to an old, decommissioned one and users get connection timeouts. Reproduce it, investigate, and fix it.**

**What is being tested:** Understanding of DNS resolution layers (resolver cache vs. authoritative answer), and how to prove a Route53 record is actually wrong — versus just a caching artifact — before touching anything.

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

aws route53 list-resource-record-sets --hosted-zone-id Z0123456789ABC \
  --query "ResourceRecordSets[?Name=='api.zen-pharma.com.']"

aws elbv2 describe-load-balancers --names old-decommissioned-nlb 2>&1
# An error occurred (LoadBalancerNotFound) — confirms the target doesn't even exist anymore
```

**Root cause:** a manual console change (or a `terraform apply` run from a stale branch whose module output still referenced the old NLB) overwrote the Route53 alias record outside the normal PR-reviewed flow, so the live record and Terraform state diverged.

**Fix:**

```bash
terraform apply -target=aws_route53_record.api   # reconciles the record back to Terraform's declared state
dig api.zen-pharma.com +short
```

<details>
<summary>📘 Teaching note</summary>

**Why `dig @8.8.8.8` matters here.** This record is an **Alias** to a load balancer, not a plain A record with a TTL you control — Route53 itself re-evaluates alias targets on every query, no caching at that layer. But resolvers *downstream* of Route53 (your laptop's OS resolver, corporate DNS, the browser) still cache whatever answer they last got, for their own TTL. Querying a public resolver directly sidesteps a possibly-stale local cache and gets you a fresh answer while you're debugging — without it, you might "fix" the record and still see the old IP for several more minutes purely from local caching, and mistakenly think the fix didn't work.

</details>

---

**Q3. Pods intermittently fail to resolve the RDS hostname — about 1 in 10 requests throw `UnknownHostException`, the rest succeed. Reproduce it, investigate, fix.**

**What is being tested:** Whether you know CoreDNS is a regular, resource-constrained Deployment that can be overloaded — not a magic always-on service — and how to prove that in-cluster.

**Break it (reproduce live):**

```bash
# Remove CoreDNS's HA and headroom
kubectl scale deployment coredns -n kube-system --replicas=1
kubectl set resources deployment coredns -n kube-system -c coredns --limits=cpu=50m,memory=30Mi

# Flood it with concurrent lookups from several pods
kubectl create job dns-stress --image=busybox --dry-run=client -o yaml \
  -- /bin/sh -c "for i in $(seq 1 5000); do nslookup drug-catalog-db.xxxxx.us-east-1.rds.amazonaws.com; done" \
  | kubectl apply -f - 
kubectl scale job dns-stress --replicas=10 2>/dev/null || kubectl create -f - <<< "$(kubectl get job dns-stress -o json)"
```

**Investigate:**

```bash
kubectl get pods -n kube-system -l k8s-app=kube-dns
kubectl top pod -n kube-system -l k8s-app=kube-dns          # check for CPU throttling / OOM proximity
kubectl logs -n kube-system -l k8s-app=kube-dns --since=5m | grep -i "error\|timeout"

kubectl exec -n prod deploy/manufacturing-service -- \
  sh -c 'for i in $(seq 1 20); do nslookup drug-catalog-db.xxxxx.us-east-1.rds.amazonaws.com; done' \
  | grep -c "can't resolve\|NXDOMAIN"

# If Prometheus scrapes CoreDNS:
#   rate(coredns_dns_responses_total{rcode="SERVFAIL"}[5m])
```

**Root cause:** CoreDNS was scaled down to a single replica (no HA) with undersized CPU/memory limits. Under query volume it gets CPU-throttled; UDP DNS queries that land during a throttled window get dropped or time out, while the rest succeed normally — hence the *intermittent*, not total, failure pattern.

**Fix:**

```bash
kubectl scale deployment coredns -n kube-system --replicas=2
kubectl set resources deployment coredns -n kube-system -c coredns \
  --limits=cpu=200m,memory=170Mi --requests=cpu=100m,memory=70Mi
```

Longer term: install the `cluster-proportional-autoscaler` for CoreDNS (scales replica count with cluster node/pod count automatically), and enable app-level DNS result caching (JVM `networkaddress.cache.ttl`, or tune `dnsConfig`/`ndots`) so not every request performs a fresh lookup.

**Verify:** rerun the 20-lookup loop from the app pod, confirm 0 failures; `kubectl top pod` shows headroom again.

---

## 2. Load Balancer & Ingress

---

**Q4. Every service behind the Ingress starts timing out at once — not just one service. Reproduce a total Ingress outage, investigate it, and fix it.**

**What is being tested:** Recognition that the Ingress controller is a *shared control plane* — one bad config change here has 100% blast radius, unlike a single bad app deploy.

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

**Root cause:** an invalid ConfigMap value broke NGINX's config template validation. Every controller replica that restarted picked up the bad config and failed to start `nginx`, taking down routing for **every** Ingress resource in the cluster simultaneously.

**Fix:**

```bash
kubectl patch configmap ingress-nginx-controller -n ingress-nginx --type merge -p \
  '{"data":{"use-forwarded-headers":"true"}}'
kubectl rollout restart deployment ingress-nginx-controller -n ingress-nginx
kubectl rollout status deployment ingress-nginx-controller -n ingress-nginx
```

**Verify:** curl several different services through the Ingress domain; confirm NLB targets go healthy again.

<details>
<summary>📘 Teaching note</summary>

Because `ingress-nginx` is a single shared dependency for every Service exposed outside the cluster, its blast radius for a bad config change is categorically different from an app-level bad deploy — a bad app deploy takes down one Service; a bad Ingress ConfigMap takes down all of them at once. This is the argument for validating ConfigMap changes with `kubectl exec <pod> -- nginx -t` before rolling them out, or running a canary replica first.

</details>

---

**Q5. `/api/v2/inventory` returns 404 through the Ingress, but every other endpoint on `drug-catalog-service` works fine. Reproduce it, investigate, fix.**

**What is being tested:** Whether you can tell an Ingress routing bug apart from an application bug — a very common trap, since both present as "404."

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

**Q6. The NLB reports all targets unhealthy, but `kubectl get pods` shows every pod `Running` and `Ready`, and the readiness probe path is confirmed correct. Reproduce it, investigate, fix.**

**What is being tested:** Whether you know to look at the network path itself — security groups, NACLs, routing — once every Kubernetes-level signal says "healthy." This is deliberately a different root cause from the classic "wrong readiness probe path" story (see `07-behavioral-realtime.md` Q5/G2) — here, everything above the network layer is fine.

**Break it (reproduce live):**

```bash
aws ec2 revoke-security-group-ingress \
  --group-id sg-0123456789nodesg \
  --protocol tcp --port 30000-32767 \
  --cidr 10.0.0.0/16
```

**Investigate:**

```bash
kubectl get pods -n prod -o wide                       # Running, Ready — all healthy
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

**Root cause:** a Terraform change to the node security group (e.g. during a compliance tightening pass) accidentally removed the ingress rule allowing the NLB's health-check probes — and real traffic — to reach the node's NodePort range. Every Kubernetes-level object (Service, Endpoints, pod readiness) is completely healthy, which is exactly why this one is hard: nothing `kubectl` can see is wrong.

**Fix:**

```bash
# revert through Terraform, not the AWS CLI directly, so state stays consistent with the fix
git revert <bad-sg-commit> && terraform apply
```

<details>
<summary>📘 Teaching note</summary>

**The general rule to state out loud in an interview:** "pods Ready, but the LB still says unhealthy" always means the problem sits *between* the load balancer and the pod, never inside either one. Check, in order: (1) Service/Endpoints, (2) security groups on the NodePort range, (3) NACLs, (4) route tables. No `kubectl` command can see this layer directly — you have to test connectivity from the network path itself (a bastion host, VPC Reachability Analyzer, or a raw `nc`/`telnet` from an instance in the same VPC).

</details>

---

## 3. Database

---

**Q7. `manufacturing-service` starts throwing `FATAL: sorry, too many clients already` from Postgres under otherwise normal traffic. Reproduce it, investigate, fix.**

**What is being tested:** Whether you check both structural causes (pool size × replica count vs. `max_connections`) and leak-style causes (idle/abandoned sessions) rather than just restarting pods and hoping.

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
psql "$RDS_ADMIN_URI" -c "SELECT usename, count(*) FROM pg_stat_activity GROUP BY usename;"

kubectl get deployment manufacturing-service -n dev -o jsonpath='{.spec.replicas}'
kubectl exec -n dev deploy/manufacturing-service -- env | grep -i hikari
# HIKARI_MAXIMUM_POOL_SIZE=10, replicas=3 → up to 30 possible connections against a lower connection ceiling
```

**Root cause:** two causes to check every time, independent of each other — (1) `replica_count × pool_size` for a single service can already exceed what the RDS instance allows, and gets worse the more services share one instance; (2) idle or abandoned sessions (a transaction opened and never closed, or a debugging `psql` session left open) hold connection slots indefinitely regardless of pool size.

**Fix:**

```bash
# Immediate relief — reclaim connections held by long-idle sessions
psql "$RDS_ADMIN_URI" -c "
  SELECT pg_terminate_backend(pid) FROM pg_stat_activity
  WHERE state = 'idle' AND state_change < now() - interval '10 minutes';"

# Structural fix — right-size the pool per replica, or add PgBouncer in front of RDS
kubectl set env deployment/manufacturing-service -n dev HIKARI_MAXIMUM_POOL_SIZE=5
```

**Verify:** `pg_stat_activity` count back under the limit; app logs clean.

---

**Q8. RDS just failed over (planned maintenance or an AZ event). The AWS console shows the failover completed in about 60 seconds, but the app keeps returning 500s for several more minutes. Reproduce it, investigate, fix.**

**What is being tested:** The distinction between *infrastructure* recovery time and *application* recovery time — a connection pool doesn't automatically know a failover happened.

**Break it (reproduce live, dev/sandbox RDS only):**

```bash
aws rds reboot-db-instance --db-instance-identifier drug-catalog-db-dev --force-failover
```

**Investigate:**

```bash
aws rds describe-events --source-identifier drug-catalog-db-dev --source-type db-instance --duration 20

kubectl logs -n dev deploy/drug-catalog-service --since=2m -f
# HikariPool-1 - Connection is not available, request timed out
# org.postgresql.util.PSQLException: An I/O error occurred while sending to the backend

psql "$RDS_ADMIN_URI" -c "SELECT 1;"     # DB itself is already healthy again — confirms the gap is app-side

kubectl exec -n dev deploy/drug-catalog-service -- env | grep -i "VALIDATION\|MAX_LIFETIME"
# HIKARI_VALIDATION_TIMEOUT=250000   (250s) ← pool keeps handing out stale connections for minutes
```

**Root cause:** HikariCP's pool has no awareness that a failover changed which physical instance the RDS endpoint DNS name points to. Existing pooled TCP connections to the old primary aren't automatically closed — they only get evicted once Hikari's own `validationTimeout`/`maxLifetime` checks catch them as dead. A window set too high means the pool keeps handing out connections it believes are healthy for minutes after the database itself has already recovered.

**Fix:**

```bash
kubectl set env deployment/drug-catalog-service -n dev HIKARI_VALIDATION_TIMEOUT=5000 HIKARI_MAX_LIFETIME=300000
kubectl rollout restart deployment drug-catalog-service -n dev
```

Add a Grafana panel on `hikaricp_connections_active` / `hikaricp_connections_timeout` so this gap is visible immediately next time instead of inferred from logs after the fact.

**Verify:** trigger another test failover in a lower environment; confirm the app recovers within seconds of the database recovering, not minutes.

---

## 4. Secrets Management

---

**Q9. `kubectl get externalsecret -A` shows `jwt-secret` in `prod` as `SecretSyncedError`. The Kubernetes Secret still exists, but pods reading it are getting stale/empty values. Reproduce it, investigate, fix.**

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

# Confirm from inside the actual IRSA identity
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

**Q10. The deploy job in GitHub Actions starts failing with `Error: Could not assume role with OIDC: Not authorized to perform sts:AssumeRoleWithWebIdentity`, right after a Terraform change to the CI IAM role. Reproduce it, investigate, fix.**

**What is being tested:** Whether you know OIDC federation does exact string matching on the JWT's `sub` claim — the same failure class as the IRSA story in `07-behavioral-realtime.md` Q6, one layer up the stack.

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

**Root cause:** the trust policy was updated to trust a different branch ref than the one actually pushing. Every workflow run on `main` presents a token whose `sub` claim the role's trust condition simply doesn't match — same exact-string-matching failure class as OIDC/IRSA anywhere else, just at the CI layer instead of the pod layer.

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

**Q11. PRs build and deploy to DEV fine. The moment a PR merges to `main`, the exact same job hangs indefinitely instead of running or failing outright. Reproduce it, investigate, fix.**

**What is being tested:** Whether you know to check the *environment/approval-gate configuration* when a job just sits there with no error — because `grep -i error` on the logs finds nothing at all.

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

**Root cause:** the job isn't hung due to a bug — it's parked at an `environment: prod` approval gate waiting for a reviewer team that was deleted (or a typo'd ID from a change to environment protection rules). No one can ever approve it, so it sits until the workflow's timeout eventually kills it.

**Fix:**

```bash
gh api -X PUT /repos/ravdy/zen-pharma-backend/environments/prod \
  -f "reviewers[][type]=Team" -F "reviewers[][id]=<real-release-managers-team-id>"
gh run rerun <run-id>
```

<details>
<summary>📘 Teaching note</summary>

This one is worth calling out specifically because it reads completely differently from every other incident in this file: nothing is throwing an error, so log-grepping finds nothing. The fix is to inspect the *approval gate configuration*, not the workflow YAML and not the job logs — a good reminder that "stuck" and "failing" are different failure modes requiring different first moves.

</details>

---

**Q12. An interviewer whose company runs Jenkins instead of GitHub Actions asks: "A Jenkins pipeline job has been stuck in the queue for 20 minutes and never starts. How do you investigate?" How do you answer this using what you actually know from the zen-pharma GitHub Actions setup?**

**What is being tested:** Whether you can generalize real experience to a tool you may not use day-to-day — a very common interviewer move to check if your knowledge is conceptual or just memorized commands.

**Concept mapping — lead with this:**

| GitHub Actions concept | Jenkins equivalent |
|---|---|
| Runner (GitHub-hosted or self-hosted) | Agent / Node |
| Job concurrency limits | Executor count per agent |
| `.github/workflows/*.yml` | Jenkinsfile |
| `environment:` approval gate | Input step / Lockable Resources |
| OIDC role trust policy | Credentials binding / cloud IAM plugin |

**Break it (if you have a real Jenkins sandbox):**

```bash
# Disconnect an agent
sudo systemctl stop jenkins-agent

# Or keep every executor busy with long-running jobs
for i in 1 2 3 4; do curl -X POST "$JENKINS_URL/job/busy-job/build" --user "$USER:$TOKEN"; done
```

**Investigate:**

```bash
curl -s "$JENKINS_URL/computer/api/json?pretty=true" --user "$USER:$TOKEN" \
  | jq '.computer[] | {displayName, offline, numExecutors}'

# The queue API tells you exactly WHY a job hasn't been assigned an executor
curl -s "$JENKINS_URL/queue/api/json?pretty=true" --user "$USER:$TOKEN" \
  | jq '.items[] | {why, stuck, blocked}'
# "why": "Waiting for next available executor on ‘agent-2’"

curl -s "$JENKINS_URL/computer/agent-2/api/json" --user "$USER:$TOKEN" | jq '.offlineCauseReason'
```

**Root cause categories to name out loud, in order of likelihood:**
1. **Agent disconnected** — network/firewall issue or an expired JNLP secret; check the agent's own logs and `systemctl status jenkins-agent`.
2. **Executors all busy** — a previous job hung with no timeout and never released its executor.
3. **Label mismatch** — the job's `agent { label 'x' }` doesn't match any connected node's labels.
4. **Disk pressure** — Jenkins refuses to schedule work on a node below its configured free-disk-space threshold.

**Fix:** reconnect the agent, abort the hung job holding the executor, correct the node label, or free disk space (Workspace Cleanup plugin / prune old builds).

<details>
<summary>📘 Teaching note</summary>

Worth including even though zen-pharma runs GitHub Actions: interviewers at Jenkins shops will ask this. The strongest answer bridges directly — "the mental model transfers 1:1: runner/agent health, concurrency limits, and approval gates are the three places I'd check on any CI system, GitHub Actions or Jenkins" — rather than saying "I haven't used Jenkins."

</details>
