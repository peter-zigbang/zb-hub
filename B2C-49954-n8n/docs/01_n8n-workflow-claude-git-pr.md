# n8n 워크플로우: Claude Code + Git + PR 자동화

> 세션 날짜: 2026-04-02 ~ 2026-04-03
> 대상 서버: n8n.zigbang.in (dev)
> 워크플로우: https://n8n.zigbang.in/workflow/3nDslsEujRxH95QG

---

## 1. 작업 목적

n8n에서 Claude Code CLI를 활용하여 **코드 수정 → 브랜치 생성 → 커밋 → 푸시 → PR 생성**까지 자동화하는 워크플로우 구축

---

## 2. 사전 작업

### 2.1 직방 소개 홈페이지 생성

- `intro.html` 단일 파일로 최초 생성 (HTML + CSS 인라인)
- 이후 `html/` 폴더로 분리: `index.html`, `style.css`, `main.js`
- JS 추가 기능: 부드러운 스크롤, 숫자 카운트업, 연혁 페이드인, 네비게이션 그림자

### 2.2 참조 워크플로우 확인

- `8ZPrXz48gpjULt5O` ([Test] Claude Code CLI 테스트) 워크플로우를 참조
- Claude Code CLI + OAuth 인증 주입 방식 확인

---

## 3. 워크플로우 구축 과정

### 3.1 초기 구조 (Anthropic API 방식)

처음에는 HTTP Request 노드로 Anthropic API를 직접 호출하는 방식으로 구현:
- 소스 파일 읽기 → HTTP Request (claude API) → 응답 파싱 → 파일 저장

**문제점**: API 키 필요, 파일을 직접 수정할 수 없음

### 3.2 Claude Code CLI 방식으로 전환

참조 워크플로우(`8ZPrXz48gpjULt5O`)와 동일한 방식으로 변경:
- OAuth 토큰을 `.credentials.json`에 주입
- `claude -p` 명령어로 직접 파일 수정

### 3.3 소스 읽기/저장 노드 제거

- 소스 파일 읽기 / 수정된 파일 저장 노드를 제거
- Claude가 `cd /tmp/n8n-repo` 에서 직접 파일 탐색 + 수정

### 3.4 커밋/푸시/PR 노드 추가

- `feature/{number}` 브랜치 생성
- `git add -A && git commit` (n8n-bot 사용자)
- `git push` → `gh pr create`

---

## 4. 트러블슈팅

### 4.1 claude -p stdin 대기 문제

```
Warning: no stdin data received in 3s, proceeding without it.
Error: Reached max turns (5)
```

**해결**: `< /dev/null` 추가로 stdin 즉시 닫기

### 4.2 n8n Execute Command exit code 에러

claude CLI가 non-zero exit code를 반환하면 n8n이 에러로 처리

**해결**: `|| true` 추가

### 4.3 git commit 실패 (user 미설정)

n8n 컨테이너에 git user.email/name 미설정

**해결**: commit 전 `git config user.email "n8n-bot@zigbang.com"` 추가

### 4.4 gh pr create --body 줄바꿈 문제

`--body` 내용에 줄바꿈이 들어가면 쉘 명령어가 깨짐

**해결**: body를 한 줄로 처리, `JSON.stringify()` 사용

### 4.5 gh auth login + GITHUB_TOKEN 충돌

컨테이너에 이미 `GITHUB_TOKEN` 환경변수가 있어서 `gh auth login` 충돌

**해결**: `GH_TOKEN={token} gh pr create ...` 환경변수 방식으로 변경

### 4.6 Webhook "Workflow could not be started" 에러

connections에 존재하지 않는 노드 참조가 남아있어서 Webhook 시작 실패

**원인**: UI에서 노드 삭제 시 connections가 남아있음
**해결**: orphan connections 정리 + 노드 재추가

### 4.7 connections 끊어짐

API로 노드 추가 후 UI에서 수정하면 connections가 리셋됨

**해결**: 매번 connections 상태 확인 후 재설정

---

## 5. 최종 워크플로우

| # | 노드 | 타입 | 명령어 / 설정 |
|---|------|------|--------------|
| - | Manual Trigger | manualTrigger | n8n UI에서 "Execute workflow" 버튼으로 실행 |
| 1.1 | Question | Set | `number`: 티켓번호, `title`: 제목, `desc`: 요청 내용 |
| 2.1 | Git 설정 | Set | `github_token`, `git_repo`: zigbang/n8n, `git_branch`: commit-test |
| 2.2 | git clone | Execute Command | `cd /tmp` |
| | | | `rm -rf n8n-repo` |
| | | | `git clone --depth 1 --branch {branch} https://x-access-token:{token}@github.com/{repo}.git n8n-repo` |
| 2.3 | branch 확인 | Execute Command | `cd /tmp/n8n-repo` |
| | | | `git branch` |
| 2.4 | git pull | Execute Command | `cd /tmp/n8n-repo` |
| | | | `git pull origin {branch}` |
| | | | `git status --short` |
| | | | `git log --oneline -5` |
| 3.1 | Claude 설정 | Set | Claude OAuth JSON |
| | | | 키체인에서 추출: `security find-generic-password -s "Claude Code-credentials" -w` |
| 3.2 | Claude 인증 주입 | Execute Command | `mkdir -p /home/node/.claude` |
| | | | `echo '{json}' > /home/node/.claude/.credentials.json` |
| 3.3 | Claude 파일 수정 | Execute Command | `cd /tmp/n8n-repo` |
| | | | `claude -p "{프롬프트}" --max-turns 10 --allowedTools Edit,Write,Read,Bash,Glob,Grep < /dev/null 2>&1 \|\| true` |
| 3.4 | 결과 확인 | Execute Command | `cd /tmp/n8n-repo` |
| | | | `git diff --stat` |
| | | | `git diff --name-only` |
| 4.1 | feature 브랜치 생성 | Execute Command | `cd /tmp/n8n-repo` |
| | | | `git checkout -b feature/{number}` |
| 4.2 | commit | Execute Command | `cd /tmp/n8n-repo` |
| | | | `git config user.email "n8n-bot@zigbang.com"` |
| | | | `git config user.name "n8n-bot"` |
| | | | `git add -A` |
| | | | `git commit -m "[{number}] {title}"` |
| 4.3 | push | Execute Command | `cd /tmp/n8n-repo` |
| | | | `git push https://x-access-token:{token}@github.com/{repo}.git feature/{number}` |
| 4.4 | PR 생성 | Execute Command | `cd /tmp/n8n-repo` |
| | | | `GH_TOKEN={token} gh pr create --title "[{number}] {title}" --body "Summary: {desc}" --base commit-test --head feature/{number} --repo {repo}` |

---

## 6. Claude Code 인증 토큰 추출

### 저장 위치

| 환경 | 저장소 |
|------|--------|
| macOS | Keychain Access (`Claude Code-credentials`) |
| Linux | libsecret / gnome-keyring |
| Docker | 키체인 없음 → `~/.claude/.credentials.json` 폴백 |

### 추출 명령어

```bash
# macOS
security find-generic-password -s "Claude Code-credentials" -w

# Linux
secret-tool lookup service "Claude Code-credentials"
```

### 주의사항

- 토큰은 만료 시간(`expiresAt`)이 있으므로 주기적으로 재추출 필요
- `refreshToken`으로 CLI가 자동 갱신하지만, n8n에 하드코딩된 값은 수동 업데이트 필요

---

## 7. n8n API 사용 명령어

### 워크플로우 조회

```bash
curl -s -H "X-N8N-API-KEY: {key}" "https://n8n.zigbang.in/api/v1/workflows/{id}"
```

### 워크플로우 업데이트

```bash
curl -s -X PUT \
  -H "X-N8N-API-KEY: {key}" \
  -H "Content-Type: application/json" \
  -d @payload.json \
  "https://n8n.zigbang.in/api/v1/workflows/{id}"
```

- `PATCH`는 지원 안 됨, `PUT` 사용
- `active` 필드는 body에서 read-only → 별도 `/activate`, `/deactivate` 사용
- body에 `staticData`, `pinData` 등 추가 필드 포함 시 "must NOT have additional properties" 에러
- 필수 필드: `name`, `nodes`, `connections`, `settings`

### 활성화/비활성화

```bash
# 활성화
curl -s -X POST -H "X-N8N-API-KEY: {key}" \
  "https://n8n.zigbang.in/api/v1/workflows/{id}/activate"

# 비활성화
curl -s -X POST -H "X-N8N-API-KEY: {key}" \
  "https://n8n.zigbang.in/api/v1/workflows/{id}/deactivate"
```

### 실행 결과 조회

```bash
curl -s -H "X-N8N-API-KEY: {key}" \
  "https://n8n.zigbang.in/api/v1/executions?workflowId={id}&limit=1&includeData=true"
```

### 전체 백업

```bash
curl -s -H "X-N8N-API-KEY: {key}" "https://n8n.zigbang.in/api/v1/workflows?limit=100"
curl -s -H "X-N8N-API-KEY: {key}" "https://n8n.zigbang.in/api/v1/credentials"
curl -s -H "X-N8N-API-KEY: {key}" "https://n8n.zigbang.in/api/v1/tags"
curl -s -H "X-N8N-API-KEY: {key}" "https://n8n.zigbang.in/api/v1/variables"
```

---

## 8. Dockerfile 참고

```
ci/docker/Dockerfile
```

- Base: `node:20-slim`
- 설치: `n8n`, `@anthropic-ai/claude-code`, `gh` (GitHub CLI), `git`, `sqlite3`
- Claude Code + gh CLI가 이미 Docker 이미지에 포함됨

---

## 9. 백업

```
/Users/jangyounc/Desktop/n8n-backup/260403_dev/
├── workflows.json
├── credentials.json
├── executions_recent.json
├── tags.json
├── variables.json
└── workflows/ (개별 15개)
```
