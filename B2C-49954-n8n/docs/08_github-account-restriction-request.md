# GitHub Account Restriction Recovery Request

> Date: 2026-04-09
> Account: peter-zigbang
> Repositories: zigbang/zigbang-client, zigbang/n8n

---

## 1. Issue

GitHub PR list not visible on web UI despite PRs existing via API.
Suspected account moderation restriction due to repetitive API activity.

## 2. Cause

n8n workflow automation + CLI(`gh`) tool caused repetitive PR create/close operations via GitHub API.

| Activity | Count |
|----------|-------|
| PRs created (zigbang/n8n) | 15 |
| PRs closed | 14 |
| GraphQL API rate limit | 0/0 (blocked) |

## 3. Recovery Request Form

### Category
- GitHub (not npm.js)

### Affected
- Username: `peter-zigbang`
- Repositories: `zigbang/zigbang-client`, `zigbang/n8n`

### Reason
- 로그인할 수 있지만 나의 프로필 및 기여가 다른 사람에게 표시되지 않음

### Subject
```
Request to Remove Account Restriction - Legitimate Development Activity
```

### Body
```
Hello GitHub Support,

I am writing to request the removal of the restriction on my account.

I am a frontend engineer at Zigbang, a proptech company based in South Korea. I use GitHub daily for legitimate development work including code collaboration, pull requests, and code reviews with my team.

Recently, I have been integrating AI-assisted development tools (n8n workflow automation) into our CI/CD pipeline, which involved creating and closing pull requests via the GitHub API. This unintentionally caused repetitive activity patterns that may have triggered GitHub's abuse detection system.

All activity was performed on our private organization repository (zigbang/n8n) for legitimate development and DevOps automation purposes. There was no intention of spam or abuse.

I have already identified the cause and will take the following steps to prevent recurrence:
- Reduce API-based PR creation frequency
- Disable automated PR creation during testing phases
- Use manual workflows instead of automated triggers for PR operations

I would greatly appreciate it if you could review my account and lift the restriction. I am happy to provide any additional information if needed.

Thank you for your time.

Best regards,
Peter Lee
Frontend Engineer, Zigbang
```

## 4. Prevention Measures

| Action | Description |
|--------|------------|
| PR creation node disable | n8n workflow test 시 4.4 PR 생성 노드 비활성화 |
| API call frequency | PR 생성/삭제 반복 작업 자제 |
| Manual workflow | 테스트 시 자동화 대신 수동 실행 |
| Rate limit monitoring | GraphQL/REST API rate limit 주기적 확인 |
