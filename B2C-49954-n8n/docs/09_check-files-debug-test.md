# a_check_files_debug Test Request

> Date: 2026-04-10
> Workflow: https://n8n.zigbang.in/workflow/5nocsCWzPdhaifaQ (a_check_files_debug)

---

## Test Payload

```json
{
  "repo": "zigbang/zigbang-client",
  "branch": "master",
  "title": "geohash를 위한 GEOHASH_ENCRYPTION_KEY 변경",
  "desc": "GEOHASH_ENCRYPTION_KEY 를 zb_test_key_for_dev 변경. 아직 키 테스트가 추가로 필요하여 이 부분을 추가 예정",
  "jira": "B2C-12345"
}
```

## Webhook

```bash
curl -X POST https://n8n.zigbang.in/webhook/a-check-files-debug \
  -H "Content-Type: application/json" \
  -d '{
    "repo": "zigbang/zigbang-client",
    "branch": "master",
    "title": "geohash를 위한 GEOHASH_ENCRYPTION_KEY 변경",
    "desc": "GEOHASH_ENCRYPTION_KEY 를 zb_test_key_for_dev 변경. 아직 키 테스트가 추가로 필요하여 이 부분을 추가 예정",
    "jira": "B2C-12345"
  }'
```
