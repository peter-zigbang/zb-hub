# n8n Backup Scheduler + Prod Config Fix

> Date: 2026-04-07
> Target: prod (n8n.zigbang.io), dev (n8n.zigbang.in)

---

## 1. Backup Workflow - Webhook → Schedule Trigger

### Prod (qQ74jAW5XkPtVvm7)

| Item | Before | After |
|------|--------|-------|
| Trigger | Webhook (POST `/backup-sqlite`) | Schedule Trigger (cron) |
| Schedule | Manual call | `30 13,19 * * *` (13:30, 19:30 KST) |
| Timezone | Not set | `Asia/Seoul` |

### Dev (ioveaPIA8ERonfTJ)

| Item | Before | After |
|------|--------|-------|
| Trigger | Webhook (POST) | Manual Trigger |

### API

```bash
# Deactivate
curl -s -X POST -H "X-N8N-API-KEY: {key}" \
  "https://n8n.zigbang.io/api/v1/workflows/{id}/deactivate"

# Update (PUT)
curl -s -X PUT -H "X-N8N-API-KEY: {key}" \
  -H "Content-Type: application/json" \
  -d @payload.json \
  "https://n8n.zigbang.io/api/v1/workflows/{id}"

# Activate
curl -s -X POST -H "X-N8N-API-KEY: {key}" \
  "https://n8n.zigbang.io/api/v1/workflows/{id}/activate"
```

---

## 2. Backup Script - Execution Purge

### Problem

- DB size after VACUUM: 49M → base64 payload: 65M
- GitHub Contents API limit: ~50MB → HTTP 422 error

### Root Cause

n8n 2.x stores execution data in two tables:
- `execution_entity` — metadata (status, time) → lightweight
- `execution_data` — node input/output data → **large** (cause of 50M+)

### Fix

Delete both tables from temp copy before upload:

```
1. sqlite3 original ".backup /tmp/copy"              ← original untouched
2. sqlite3 /tmp/copy "DELETE FROM execution_entity;"  ← purge from copy
3. sqlite3 /tmp/copy "DELETE FROM execution_data;"    ← purge from copy
4. sqlite3 /tmp/copy "VACUUM;"                        ← compress
5. base64 → GitHub API PUT                            ← upload
6. rm -f /tmp/copy                                    ← cleanup
```

### Result

| Item | Before | After |
|------|--------|-------|
| Raw DB | 121M | 121M (untouched) |
| After purge+VACUUM | 51M | **1.2M** |
| Payload size | 68M | **1.5M** |
| GitHub API | HTTP 422 | **HTTP 200** |

### Stats Output

```
Step 0 OK: purged executions from temp copy (had 245 rows)
Step 1 OK: backup 121M -> purge+VACUUM -> 1.2M
Stats: workflows:9 credentials:16 executions(original):245 variables:0 datatables:0 users:9
Step 2 OK: SHA=...
Step 2.5: payload size=1.5M
Step 3 OK: Committed (20260407_130027) HTTP 200
```

---

## 3. Script Deployment via n8n API

### Shell heredoc issue

Shell `${VAR}` substitution inside heredoc caused base64 not to update. Fixed by building payload with Python:

```python
import json, base64
b64_script = base64.b64encode(script.encode()).decode()
payload = { "nodes": [...], "connections": {...} }
with open('/tmp/payload.json', 'w') as f:
    json.dump(payload, f)
```

### Verification

```bash
curl -s -H "X-N8N-API-KEY: {key}" \
  "https://n8n.zigbang.io/api/v1/workflows/{id}" | python3 -c "
import sys, json, base64
d = json.load(sys.stdin)
cmd = [n for n in d['nodes'] if n['name'] == 'Backup & Commit to GitHub'][0]['parameters']['command']
# extract and decode base64 to verify script content
"
```

---

## 4. Prod Config Fix

### env/.env.prod

| Item | Before | After |
|------|--------|-------|
| REGION | ap-northeast-1 | **ap-northeast-2** |
| TASK_ROLE | n8n-task-role | **n8n-prod-task-role** |
| EXECUTION_ROLE | n8n-execution-role | **n8n-prod-execution-role** |

### ci/azure-pipelines.yml

| Item | Before | After |
|------|--------|-------|
| AWS_REGION | dev/prod branch conditional | Fixed `ap-northeast-2` |

### Naming Pattern (unified)

| Item | Dev | Prod |
|------|-----|------|
| Region | ap-northeast-2 | ap-northeast-2 |
| Task Role | n8n-dev-task-role | n8n-prod-task-role |
| Execution Role | n8n-dev-execution-role | n8n-prod-execution-role |

---

## 5. PR #10 Conflict Resolution

- PR: `master → deploy/prod`
- Conflicts: 3 files

| File | Resolution |
|------|-----------|
| `backups/n8n_db.sqlite` | Deleted (replaced by dev/prod split files) |
| `ci/azure-pipelines.yml` | master version (AWS_REGION fixed) |
| `env/.env.prod` | master version (actual AWS config) |

Commit: `8b5350f merge: master → deploy/prod`

---

## 6. AWS Infra Summary

### ECS

| Item | Dev | Prod |
|------|-----|------|
| Domain | n8n.zigbang.in | n8n.zigbang.io |
| Region | ap-northeast-2 (Seoul) | ap-northeast-2 (Seoul) |
| AWS Account | 768556645518 | 768556645518 |
| Fargate | Spot 100% | On-Demand 100% |
| CPU | 512 (0.5 vCPU) | 1024 (1 vCPU) |
| Memory | 1024 MB | 2048 MB |

### CI/CD

| Item | Dev | Prod |
|------|-----|------|
| Branch | deploy/dev | deploy/prod |
| Seed DB | backups/n8n_db_dev.sqlite | backups/n8n_db_prod.sqlite |

### Backup

| Item | Dev | Prod |
|------|-----|------|
| Trigger | Manual | Schedule (13:30, 19:30 KST) |
| File | backups/n8n_db_dev.sqlite | backups/n8n_db_prod.sqlite |

---

## 7. Route 53 Region Migration Verification

n8n.zigbang.io DNS record changed from Tokyo to Seoul.

```
$ dig n8n.zigbang.io +short
3.37.227.203   → ap-northeast-2 (Seoul)
54.180.190.88  → ap-northeast-2 (Seoul)

$ curl -s -o /dev/null -w "HTTP %{http_code}" https://n8n.zigbang.io/healthz
HTTP 200
```

---

## 8. Old AWS Resource Cleanup Request

### Target (to delete)

```
[Region: ap-northeast-1 (Tokyo)]
- CloudFormation Stack: n8n related stacks
- ECS Cluster / Service / Task Definition
- ECR Repository
- ALB / Target Group
- CloudWatch Log Group

[IAM (Global)]
- n8n-task-role       → replaced by n8n-prod-task-role
- n8n-execution-role  → replaced by n8n-prod-execution-role
```

- AWS Account: `768556645518`
- CloudFormation stack deletion will clean up most related resources
- IAM Roles are global and may remain after stack deletion

---

## 9. GitHub Branch Protection

Branch protection rules set for deploy branches.

| Branch | Direct push | PR merge | Required approvals | Self approve |
|--------|------------|----------|-------------------|-------------|
| `deploy/dev` | Blocked | Allowed | 0 | Allowed |
| `deploy/prod` | Blocked | Allowed | 0 | Allowed |
| `master` | Allowed (no rule) | Allowed | - | - |

### Settings

- `enforce_admins: false` — admin can bypass if needed
- `dismiss_stale_reviews: false`
- `require_code_owner_reviews: false`

### API

```bash
# Set branch protection
gh api repos/zigbang/n8n/branches/{branch}/protection \
  -X PUT --input - << 'EOF'
{
  "required_pull_request_reviews": {
    "required_approving_review_count": 0,
    "dismiss_stale_reviews": false,
    "require_code_owner_reviews": false
  },
  "required_status_checks": null,
  "enforce_admins": false,
  "restrictions": null
}
EOF
```

---

## 10. Project Audit Summary

Full audit of `/n8n` directory identified 25 issues.

### Critical Issues

| # | File | Issue |
|---|------|-------|
| 1 | env/.env.dev, .env.prod | Slack Webhook URL hardcoded in source |
| 2 | env/.env.local | N8N_ENCRYPTION_KEY in plaintext |
| 3 | ci/cdk/app.ts:46-47 | Secrets passed as plaintext ECS env vars (isSsm: false) |
| 4 | ci/cdk/app.ts:59-70 | IAM wildcard: actions ["*"], resources ["*"] |
| 5 | backup-watcher.sh:11 | GITHUB_BRANCH hardcoded to deploy/dev |

### High Issues

| # | File | Issue |
|---|------|-------|
| 6 | Dockerfile:26 | COPY backups/n8n_db.sqlite — file doesn't exist directly |
| 7 | backup-watcher.sh | Uses inotifywait but inotify-tools not installed |

### Pending Actions

1. Slack webhook, encryption key → Azure KeyVault / SSM
2. IAM policies → least privilege
3. backup-watcher branch → use env variable
4. inotify-tools → add to Dockerfile or remove watcher
5. entrypoint.sh → clean up unused cron/watcher code
