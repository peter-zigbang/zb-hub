# B2C-49770: [FE] 위치정보 암호화 및 전송

## 에픽 정보
- **상위 에픽:** B2C-49607 — [BE] Geohash 암호화 — 위치정보법 제16조 대응
- **담당자:** Peter Lee (jangyoun@zigbang.com)
- **상태:** 구현 완료 (코드리뷰 대기)

## 배경
위치정보법 제16조 대응 — 클라이언트-서버 간 위치 파라미터 암호화 전송 필요.

## FE 작업 범위 (2가지)

### 1. Geohash 암복호화 (대칭키)
- **알고리즘:** AES-256-CBC + HMAC-derived IV (결정적 암호화)
- **특징:** 동일 입력 → 동일 암호문 (CDN 캐시 유지)
- **환경변수:** `GEOHASH_ENCRYPTION_KEY` (base64 인코딩된 32바이트)
- **BE 상태:** house, apis-v2, apt-api 모두 복호화 인터셉터 배포 완료
- **하위호환:** 키 미설정 시 경고 로그 + 복호화 스킵

### 2. "내 위치찾기" 좌표 암호화 (비대칭키)
- **알고리즘:** RSA-OAEP (SHA-256)
- **키:** Ed가 전달한 PEM 공개키 (`my-position-only.pem`)
- **입력:** `JSON.stringify({ lat, lng })` (~40바이트, RSA_2048 제한 190바이트 이내)
- **출력:** base64 문자열 (~342자)
- **암호화 주체:** FE (Web Crypto API)
- **복호화:** BE — KMS DecryptCommand (EncryptionAlgorithm: RSAES_OAEP_SHA_256)
- **SDK:** `common-zigbang-api-sdk-v0.4.73`
- **API:** https://apis.zigbang.net/common/api/#/Index/IndexController_saveLocation
- **BE 상태:** dev 환경 실제 저장 가능하도록 배포 완료

## 하위 티켓 전체 현황

| # | 티켓 | 제목 | 담당자 | 상태 |
|---|---|---|---|---|
| 1 | B2C-49608 | [BE] Geohash 암호화 키 등록 (env) | Jeremy Jang | 완료 |
| 2 | B2C-49609 | [BE] Geohash 복호화 — house | Jeremy Jang | 완료 |
| 3 | B2C-49610 | [BE] Geohash 복호화 — zigbang-apis-v2 | Jeremy Jang | 완료 |
| 4 | B2C-49611 | [BE] 통합 테스트 및 배포 | Jeremy Jang | 완료 |
| 5 | B2C-49635 | [BE] apt 레포 geohash 암/복호화 | Ed (홍지훈) | 코드리뷰 |
| 6 | B2C-49636 | [BE] common-api에 내 위치보기 API 신설 | Ed (홍지훈) | 코드리뷰 |
| 7 | B2C-49770 | [FE] 위치정보 암호화 및 전송 | Peter Lee | 진행중 |

## 배포 순서
서버 먼저 배포 → 클라이언트 나중 (하위호환 보장)

## 슬랙 논의 (2026-03-30 ~ 04-02)
- Ed: apt-api geohash 복호화 인터셉터 dev 배포 완료
- Jeff: 비대칭키 암호화 구현 방안 정리 (KMS vs 자체 RSA)
- Ed: 공개키 전달 + 신규 API dev 배포 (noop → 실제 저장 가능)
- Ed: SDK `common-zigbang-api-sdk-v0.4.73` 사용 안내
- Ed: 프리뷰 테스트 일정 문의 (4/2)
- Peter: 오늘 확인 후 일정 공유 예정
