# B2C-49607 위치정보 암호화 — Architecture

## 1. 개요

위치정보법 제16조 대응을 위해 클라이언트-서버 간 위치 파라미터를 암호화하여 전송한다.

## 2. 암호화 프로토콜

### 2.1 Geohash 암호화 (대칭키)

```
┌─ Client (FE) ──────────────────────────────────────┐
│                                                     │
│  geohash (plaintext, e.g. "wydm9")                 │
│       │                                             │
│       ▼                                             │
│  HMAC-SHA256(geohash, key) → IV (앞 16바이트)       │
│       │                                             │
│       ▼                                             │
│  AES-256-CBC(geohash, key, iv) → ciphertext        │
│       │                                             │
│       ▼                                             │
│  base64(IV + ciphertext) → encrypted geohash       │
│       │                                             │
│       ▼                                             │
│  axios request → params.geohash = encrypted        │
│                                                     │
└──────────────────┬──────────────────────────────────┘
                   │ HTTPS
                   ▼
┌─ Server (BE) ──────────────────────────────────────┐
│                                                     │
│  복호화 인터셉터 (house, apis-v2, apt-api)          │
│       │                                             │
│       ▼                                             │
│  base64 decode → IV (앞 16바이트) + ciphertext     │
│       │                                             │
│       ▼                                             │
│  AES-256-CBC decrypt(ciphertext, key, iv)          │
│       │                                             │
│       ▼                                             │
│  geohash (plaintext) → 기존 로직 그대로 처리        │
│                                                     │
└─────────────────────────────────────────────────────┘
```

**핵심 결정사항:**
- 결정적 암호화 (동일 입력 → 동일 암호문): CDN 캐시 유지 목적
- IV를 HMAC에서 파생: 랜덤 IV 대신 결정적 IV → 같은 geohash는 같은 암호문
- 하위호환: `GEOHASH_ENCRYPTION_KEY` 미설정 시 평문 전송 (서버 먼저 배포 가능)

### 2.2 "내 위치찾기" 암호화 (비대칭키)

```
┌─ Client (FE) ──────────────────────────────────────┐
│                                                     │
│  GPS 좌표: { lat: 37.4979, lng: 127.0276 }         │
│       │                                             │
│       ▼                                             │
│  JSON.stringify({ lat, lng }) → ~40바이트           │
│       │                                             │
│       ▼                                             │
│  RSA-OAEP(plaintext, publicKey, SHA-256)            │
│       │  (Web Crypto API)                           │
│       ▼                                             │
│  base64(encrypted) → ~342자                         │
│       │                                             │
│       ▼                                             │
│  POST /v2/my-position { encryptedLocation }         │
│                                                     │
└──────────────────┬──────────────────────────────────┘
                   │ HTTPS
                   ▼
┌─ Server (BE) ──────────────────────────────────────┐
│                                                     │
│  KMS DecryptCommand                                 │
│  (EncryptionAlgorithm: RSAES_OAEP_SHA_256)        │
│       │                                             │
│       ▼                                             │
│  { lat, lng } → 위치 저장                           │
│                                                     │
└─────────────────────────────────────────────────────┘
```

**핵심 결정사항:**
- 비대칭키: FE는 공개키만 보유, 복호화 불가 → 보안 강화
- KMS 연동: BE에서 AWS KMS로 복호화 → 키 관리 중앙화
- 실패 무시: 암호화 전송 실패 시 기존 기능(지도 이동)에 영향 없음

## 3. 키 관리

| 키 | 유형 | 저장 위치 | 관리 |
|---|---|---|---|
| `GEOHASH_ENCRYPTION_KEY` | 대칭 (AES-256) | 환경변수 (base64, 32바이트) | BE/FE 공유 |
| `MY_POSITION_PUBLIC_KEY` | 비대칭 (RSA 공개키) | 환경변수 (PEM) | Ed 전달, FE only |
| RSA 개인키 | 비대칭 (RSA 개인키) | AWS KMS | BE only |

## 4. 영향 범위

### FE 레포

| 레포 | 영향 | 설명 |
|------|------|------|
| zigbang-client | Geohash + RSA | 5개 맵 화면, axios interceptor |
| single-page | postMessage 보안 | origin 제한 |

### BE 레포

| 레포 | 영향 | 담당 | 상태 |
|------|------|------|------|
| house | Geohash 복호화 | Jeremy | 완료 |
| zigbang-apis-v2 | Geohash 복호화 | Jeremy | 완료 |
| apt-api | Geohash 복호화 | Ed | 코드리뷰 |
| common-api | 내 위치보기 API | Ed | 코드리뷰 |

## 5. 배포 전략

```
Phase 1: BE 복호화 인터셉터 배포 (완료)
  → 평문/암호문 모두 처리 가능 (하위호환)

Phase 2: FE 암호화 배포
  → 환경변수 설정 후 암호문 전송 시작

Phase 3: 평문 차단 (선택)
  → BE에서 평문 요청 거부 (강제 적용)
```
