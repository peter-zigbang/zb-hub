# deploy/dev Cherry-pick 및 담당자 공지 작성

> 날짜: 2026-04-06  
> 브랜치: deploy/dev, deploy/prod  
> 작업: prod 전용 커밋을 dev에 반영 + 복구 공지 퇴고

---

## 1. deploy/dev에 누락된 커밋 반영

### 배경
`deploy/prod`에만 반영된 2개 커밋이 `deploy/dev`에 누락되어 있었음.

| 커밋 | 내용 |
|------|------|
| `943a8b9` | 환경별 리전 분리: dev(서울), prod(도쿄) |
| `fb2b32d` | IAM Role 이름을 환경변수로 분리 |

### 확인
```bash
git branch -r --contains 943a8b9  # origin/deploy/prod만 출력
git branch -r --contains fb2b32d  # origin/deploy/prod만 출력
```

### 작업
```bash
git checkout deploy/dev
git cherry-pick 943a8b9 fb2b32d
```

### 충돌 해결
- `backups/n8n_db.sqlite` 바이너리 충돌 발생
- dev 환경의 sqlite를 유지 (`git checkout --ours`)하고 cherry-pick 계속 진행

### 결과
```
42285b2 환경별 리전 분리: dev(서울), prod(도쿄)
bdd3b78 IAM Role 이름을 환경변수로 분리
```

---

## 2. 복구 공지 퇴고

422 에러로 인한 workflow/credentials 누락 관련 담당자 공지 문구를 정리.

### 최종 공지문

> **n8n 배포 과정에서 일부 workflow와 credentials이 누락되어 담당자 확인을 요청드립니다.**
>
> **배경**  
> n8n 데이터를 git commit으로 백업하고 있는데, commit 시 422 에러가 발생하여 수정 코드를 반영/배포하였습니다.
>
> **현재 상황**  
> 오류 기간 중 생성된 workflow와 credentials은 직접 복구하였으나, 아래 두 가지 이슈가 있습니다.
> 1. **Credentials** - 암호화되어 있어 원본 복구가 불가하여 임의의 값으로 대체한 상태. 재설정 필요.
> 2. **Workflow** - 일부 누락이 있을 수 있음.
>
> 해당 workflow/credentials 담당자분들께서 정상 동작 여부를 확인해 주시면 감사하겠습니다.
