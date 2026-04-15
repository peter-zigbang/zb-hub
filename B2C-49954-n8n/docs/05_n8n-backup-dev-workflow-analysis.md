# n8n backup_dev 워크플로우 분석

> 날짜: 2026-04-06  
> 환경: Dev (https://n8n.zigbang.in)  
> 워크플로우: https://n8n.zigbang.in/workflow/ioveaPIA8ERonfTJ  
> 소유자: Jang Youn (jhso)  
> 상태: active

---

## 1. 개요

n8n의 SQLite DB를 GitHub에 직접 커밋하는 백업 워크플로우.  
git clone/push 없이 **GitHub Contents API**로 직접 커밋하는 방식.

---

## 2. 플로우 구조

```
Webhook (POST) → Config (설정) → Backup & Commit to GitHub (실행)
```

| # | 노드 | 타입 | 설명 |
|---|------|------|------|
| 1 | Webhook | webhook (v2) | POST `/backup-sqlite` 수신 시 트리거 |
| 2 | Config | set (v3.4) | GitHub 토큰, 브랜치, 파일경로 설정 |
| 3 | Backup & Commit to GitHub | executeCommand (v1) | base64 인코딩된 쉘 스크립트 실행 |

### Webhook URL

```
https://n8n.zigbang.in/webhook/backup-sqlite
```

### Config 설정값

| 변수 | 값 |
|------|-----|
| `github_token` | GitHub PAT |
| `github_branch` | `master` |
| `github_file_path` | `backups/n8n_db_dev.sqlite` |

---

## 3. 쉘 스크립트 상세 (base64 디코딩)

Execute Command 노드에서 base64로 인코딩된 쉘 스크립트를 디코딩 후 실행.

### 환경변수

| 변수 | 값 / 소스 |
|------|----------|
| `GITHUB_REPO` | `zigbang/n8n` (하드코딩) |
| `BRANCH` | Config 노드의 `github_branch` |
| `FILE_PATH` | Config 노드의 `github_file_path` |
| `DB` | `/home/node/.n8n/database.sqlite` |
| `TOKEN` | Config 노드의 `github_token` |
| `TIMESTAMP` | `date +"%Y%m%d_%H%M%S"` |
| `ENV_LABEL` | 파일경로에서 추출 (`n8n_db_dev.sqlite` → `dev`) |

### Step 1 - DB 백업 + VACUUM

```bash
# DB 통계 수집
CRED_CNT=$(sqlite3 "$DB" "SELECT COUNT(*) FROM credentials_entity;")
WF_CNT=$(sqlite3 "$DB" "SELECT COUNT(*) FROM workflow_entity;")
USER_CNT=$(sqlite3 "$DB" "SELECT COUNT(*) FROM user;")

# 안전한 백업 (.backup 명령어 사용)
sqlite3 "$DB" ".backup $BACKUP"

# VACUUM으로 파일 크기 최적화
sqlite3 "$BACKUP" "VACUUM;"
```

- `/home/node/.n8n/database.sqlite` → `/tmp/db_backup_{timestamp}.sqlite`
- `.backup` 명령어로 안전한 복사 (실행 중 DB에도 안전)
- VACUUM으로 사이즈 축소
- credentials, workflows, users 카운트 수집

### Step 2 - GitHub 기존 파일 SHA 조회

```bash
SHA_RESP=$(curl -s \
  -H "Authorization: token ${TOKEN}" \
  -H "Accept: application/vnd.github.v3+json" \
  "https://api.github.com/repos/${GITHUB_REPO}/contents/${FILE_PATH}?ref=${BRANCH}")
SHA=$(echo "$SHA_RESP" | grep -o '"sha": *"[^"]*"' | head -1 | sed 's/"sha": *"//;s/"//')
```

- 기존 파일이 있으면 SHA 추출 (업데이트 시 필수)
- 없으면 SHA 비어있음 → 새 파일 생성

### Step 3 - GitHub Contents API로 커밋

```bash
# payload 생성: message + base64 content + sha(있으면) + branch
printf '{"message":"%s","content":"' "$COMMIT_MSG" > "$PF"
base64 "$BACKUP" | tr -d '\n' >> "$PF"

# GitHub Contents API PUT
HTTP=$(curl -s -o "$RF" -w "%{http_code}" -X PUT \
  -H "Authorization: token ${TOKEN}" \
  -H "Accept: application/vnd.github.v3+json" \
  "https://api.github.com/repos/${GITHUB_REPO}/contents/${FILE_PATH}" \
  -d @"$PF")
```

- sqlite 파일을 base64 인코딩하여 JSON payload 생성
- `PUT /repos/zigbang/n8n/contents/backups/n8n_db_dev.sqlite`로 직접 커밋
- 커밋 메시지 형식: `[n8n-dev] backup: 20260406_091327 | credentials:17 workflows:15 users:9`
- HTTP 200/201 → 성공, 그 외 → 실패 + 에러 응답 출력
- 임시 파일 정리 (`rm -f`)

---

## 4. 특징

| 항목 | 설명 |
|------|------|
| 커밋 방식 | GitHub Contents API (git clone/push 없음) |
| 장점 | 가볍고 빠름, git 설정 불필요 |
| 제약 | GitHub API 파일 크기 제한 (100MB) |
| 환경 구분 | 파일경로에서 env label 자동 추출 |
| 에러 핸들링 | 각 Step별 실패 시 에러 메시지 + exit 1 |
| 정리 | 임시 파일(backup, payload, response) 모두 삭제 |

---

## 5. 대상 리포 & 브랜치

- **리포**: `zigbang/n8n`
- **브랜치**: `master`
- **파일**: `backups/n8n_db_dev.sqlite`
