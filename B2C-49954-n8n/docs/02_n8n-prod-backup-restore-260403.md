# n8n Prod 백업 복원 (260403 백업 기준)

> 날짜: 2026-04-06  
> 환경: Prod (https://n8n.zigbang.io)  
> 백업 소스: `/Users/jangyounc/Desktop/n8n-backup/260403`  
> API Key: prod 서버 (X-N8N-API-KEY 헤더)

---

## 1. 유저 비교

백업(260403)과 Prod 현재 유저 목록 **동일 (9명)**, ID도 일치.  
차이점은 `updatedAt` 타임스탬프뿐:

| 이름 | 이메일 | 백업(260403) | Prod(현재) | 비고 |
|------|--------|-------------|-----------|------|
| Glen Lee | gr8woo@zigbang.com | 03-31 | 02-23 | 백업이 더 최신 |
| Aron Lee | wook@zigbang.com | 04-01 | 03-23 | 백업이 더 최신 |
| sam im | syim@zigbang.com | 04-02 | 02-23 | 백업이 더 최신 |
| jhso zigbang | jhso@zigbang.com | 04-02 | 04-02 | 동일 |
| 나머지 5명 | — | 동일 | 동일 | — |

---

## 2. 크리덴셜 복원

### 복원 결과

| # | 이름 | 타입 | 소유자 | 상태 |
|---|------|------|--------|------|
| 1 | Google Gemini(PaLM) Api account | googlePalmApi | Jang Youn (jhso) | 기존 |
| 2 | Jira SW Cloud account | jiraSoftwareCloudApi | Jang Youn (jhso) | 기존 |
| 3 | Slack account | slackApi | Jang Youn (jhso) | 기존 |
| 4 | GitHub account | githubApi | Jang Youn (jhso) | 기존 |
| 5 | Google Service Account account | googleApi | Jang Youn (jhso) | 기존 |
| 6 | Jira SW Cloud account 2 | jiraSoftwareCloudApi | Aron Lee (wook) | fake |
| 7 | Slack account 2 | slackApi | Aron Lee (wook) | fake |
| 8 | GitHub account 3 | githubApi | Aron Lee (wook) | fake |
| 9 | Google Gemini(PaLM) Api account 2 | googlePalmApi | Aron Lee (wook) | fake |
| 10 | Header Auth account | httpHeaderAuth | Aron Lee (wook) | fake |
| 11 | Google Calendar account | googleCalendarOAuth2Api | Glen Lee (gr8woo) | fake |
| 12 | Slack account 3 | slackOAuth2Api | Glen Lee (gr8woo) | fake |
| 13 | Notion API (FE 주간회의) | httpHeaderAuth | Glen Lee (gr8woo) | fake |
| 14 | github_peter | githubApi | Jang Youn (jhso) | fake |
| 15 | azure-pat | httpBearerAuth | sam im (syim) | fake |
| 16 | hono | httpBearerAuth | sam im (syim) | fake |
| 17 | GitHub account | githubApi | sam im (syim) | fake |

### 소유자 이관 방법

- n8n API `PUT /credentials/{id}/transfer` 사용
- `destinationProjectId`에 백업의 `shared[].id` (personal project ID) 전달
- 예: Glen Lee의 projectId = `F4qXpIMs8fvWfj8V`

### 주의사항

- fake 값으로 생성된 12개 크리덴셜은 각 소유자가 n8n UI에서 실제 시크릿 입력 필요
- 크리덴셜 ID가 백업과 다름 (API 생성 시 새 ID 발급)

---

## 3. 워크플로우 복원

### 복원 결과

| # | 워크플로우 | 소유자 | 상태 | 특이사항 |
|---|-----------|--------|------|---------|
| 1 | My workflow | Glen Lee (gr8woo) | 기존 | - |
| 2 | Jira to Slack from Aron | Aron Lee (wook) | 기존 | - |
| 3 | SDD | Jang Youn (jhso) | 기존 | - |
| 4 | FE_APT_WORKFLOW | Jang Youn (jhso) | 기존 | - |
| 5 | PR-Review | sam im (syim) | 복구 | fake 크리덴셜 (hono, GitHub account), settings 일부 제거 |
| 6 | shedule-app-build-failed | sam im (syim) | 복구+활성화 | fake 크리덴셜 (hono, azure-pat), **staticData 유실 — 첫 실행 시 중복 알림 가능**, settings 일부 제거 |
| 7 | FE 주간회의 — 스킵 판단 + 캘린더 | Glen Lee (gr8woo) | 복구+활성화 | fake 크리덴셜 (Notion API), staticData 유실 (영향 낮음) |
| 8 | FE 주간회의 — 회의록 자동 생성 | Glen Lee (gr8woo) | 복구+활성화 | fake 크리덴셜 (Notion API), staticData 유실 (영향 낮음) |

### 삭제된 워크플로우

| 워크플로우 | 사유 |
|-----------|------|
| test01 (RtB5GP9HmhDFbrl0) | 불필요 |
| Download SQLite DB (UbaxyxUkwij9ofUf) | 불필요 |
| DB Backup to GitHub (A09wJm2bAZj3X4GP) | 대체됨 |
| DB Backup to GitHub API (Cwt6QUaZ334xvVNj) | 대체됨 |

### 새로 이관된 워크플로우

| 워크플로우 | 소스 | 변경사항 |
|-----------|------|---------|
| backup_wf_by_git (qQ74jAW5XkPtVvm7) | dev → prod | `github_branch`: deploy/dev → deploy/prod |

### 소유자 이관 방법

- `PUT /workflows/{id}/transfer` + `{"destinationProjectId": "..."}` 사용
- 크리덴셜과 동일한 personal project ID 사용

### 복원 시 제거된 settings

| 설정 | 사유 |
|------|------|
| `availableInMCP` | n8n API 스키마 미지원 (전체 7개 복원 워크플로우) |
| `binaryMode` | n8n API 스키마 미지원 (PR-Review, DB Backup to GitHub, shedule-app-build-failed) |

---

## 4. dev 워크플로우 Webhook 변경

### 대상
- `3nDslsEujRxH95QG` (git clone 및 commit 테스트) on dev (n8n.zigbang.in)

### 변경 내용
- Manual Trigger → **Webhook** (POST, path: `git-clone-commit`)
- `1.1 Question` 노드: 변경 없음 (하드코딩 유지)
- Webhook URL: `https://n8n.zigbang.in/webhook/git-clone-commit`

### 실행 에러 분석

**근본 원인**: `3.1 Claude 설정` 노드의 OAuth 토큰 만료
```
OAuth token has expired. Please obtain a new token or refresh your existing token.
```

**결과**: Claude 파일 수정이 실행 안 됨 → 변경사항 없음 → `4.2 commit`에서 `nothing to commit` 에러

**해결 필요**: Claude OAuth 토큰 (`accessToken`, `refreshToken`) 갱신

---

## 5. API 참고사항

### Personal Project ID 매핑 (소유자 이관용)

| 소유자 | projectId |
|--------|-----------|
| Jang Youn (jhso) | `ohjRZL4Uki7xschM` |
| Glen Lee (gr8woo) | `F4qXpIMs8fvWfj8V` |
| Aron Lee (wook) | `wrD3XiEgW22hxJit` |
| sam im (syim) | `J2dw27HPCJbV9mHM` |
| peter lee (jangyoun) | `a67hpFZ93FdYW3yr` |

### API 키

- **dev** (n8n.zigbang.in): 별도 키 사용
- **prod** (n8n.zigbang.io): 별도 키 사용
- 키는 `X-N8N-API-KEY` 헤더로 전달

### 크리덴셜 생성 시 주의

- OAuth2 타입은 `serverUrl`, `sendAdditionalBodyProperties`, `additionalBodyProperties` 필드 필수
- `slackApi` 타입은 `notice` 필드 필수
