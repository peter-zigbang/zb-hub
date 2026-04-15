# n8n 환경별 리전 분리 및 DB 초기화

- 날짜: 2026-04-06
- 브랜치: `deploy/prod`
- 관련 커밋: `943a8b9`, `fb2b32d`

## 작업 요약

### 1. n8n DB 초기화 (SQLite)

n8n 인스턴스를 초기 상태로 리셋하기 위해 `backups/n8n_db.sqlite`의 모든 테이블 데이터를 삭제했다.

- 대상: `backups/n8n_db.sqlite`
- `migrations` 테이블은 유지 (n8n 재시작 시 스키마 호환성 보장)
- 나머지 모든 테이블 `DELETE` 처리
- 주요 삭제 데이터: credentials(5), workflows(5), executions(267), users(9), settings(7)

### 2. 환경별 리전 분리

기존에는 리전이 `ap-northeast-2`(서울)로 하드코딩되어 있어 prod(도쿄) 배포 시 잘못된 리전으로 배포되는 문제가 있었다.

| 환경 | 리전 | 도메인 |
|------|------|--------|
| dev | ap-northeast-2 (서울) | n8n.zigbang.in |
| prod | ap-northeast-1 (도쿄) | n8n.zigbang.io |

#### 변경 파일

| 파일 | 변경 내용 |
|------|----------|
| `env/.env.dev` | `REGION=ap-northeast-2` 추가 |
| `env/.env.prod` | `REGION=ap-northeast-1` 추가 |
| `ci/cdk/cdk.ts` | `region`을 `process.env.REGION`에서 읽도록 변경 |
| `ci/azure-pipelines.yml` | `AWS_REGION` 변수 추가, OIDC 인증 리전 동적 설정 |

### 3. IAM Role 이름 환경변수 분리

도쿄 리전 배포 시 IAM Role 충돌 발생 (`AlreadyExists`). `deploy/test` 브랜치가 이미 도쿄 리전에 `n8n-task-role`, `n8n-execution-role`을 생성해둔 상태였다.

IAM Role은 글로벌 리소스이므로 리전이 달라도 같은 이름의 Role은 생성 불가.

#### 해결

Role 이름을 env 파일에서 환경변수로 관리하도록 분리:

| 환경 | TASK_ROLE | EXECUTION_ROLE |
|------|-----------|----------------|
| dev | `n8n-dev-task-role` | `n8n-dev-execution-role` |
| prod | `n8n-task-role` | `n8n-execution-role` |

#### 변경 파일

| 파일 | 변경 내용 |
|------|----------|
| `env/.env.dev` | `TASK_ROLE`, `EXECUTION_ROLE` 추가 |
| `env/.env.prod` | `TASK_ROLE`, `EXECUTION_ROLE` 추가 |
| `ci/cdk/app.ts` | `process.env.TASK_ROLE`, `process.env.EXECUTION_ROLE`로 변경 |

## 트러블슈팅

### CloudFormation IAM Role 충돌

- 에러: `Resource of type 'AWS::IAM::Role' with identifier 'n8n-prod-execution-role' already exists`
- 원인: `deploy/test` 브랜치에서 도쿄 리전에 stage 없는 이름(`n8n-task-role`)으로 Role을 생성했으나, `deploy/prod`에서는 `n8n-prod-task-role`로 생성 시도 -> 기존 Role과 충돌
- 해결: prod의 Role 이름을 `deploy/test`와 동일하게 맞추고, 환경변수로 분리하여 관리

### IAM Role 삭제 불가

- 서울 리전의 CloudFormation 스택 삭제 후에도 IAM Role이 남아있는 경우 발생
- IAM Role 삭제 권한이 없어 수동 삭제 불가
- 기존 Role 이름을 재사용하는 방향으로 우회 해결
