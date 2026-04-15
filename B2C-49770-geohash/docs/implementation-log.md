# B2C-49770 구현 로그

## 작업일: 2026-04-02

### 변경 파일 목록

#### zigbang-client-ro — 신규 (3개)

| 파일 | 설명 |
|------|------|
| `packages/screens/src/lib/crypto/geohashCrypto.ts` | AES-256-CBC + HMAC-derived IV 결정적 암호화 |
| `packages/screens/src/lib/crypto/locationCrypto.ts` | RSA-OAEP (SHA-256) Web Crypto API 암호화 |
| `packages/screens/src/lib/crypto/sendEncryptedLocation.ts` | 암호화된 좌표 POST 전송 래퍼 |

#### zigbang-client-ro — 수정 (6개)

| 파일 | 변경 내용 |
|------|-----------|
| `packages/screens/src/lib/House/axios/index.ts` | request interceptor에 geohash 암호화 로직 추가 |
| `packages/screens/src/home/oneroom/HomeOneroomMapScreen/index.tsx` | import + sendEncryptedLocation 호출 |
| `packages/screens/src/home/officetel/HomeOfficetelMapScreen/index.tsx` | 동일 |
| `packages/screens/src/home/villa/HomeVillaMapScreen/index.tsx` | 동일 |
| `packages/screens/src/home/apt/HomeAptMapScreen/index.tsx` | 동일 |
| `packages/screens/src/home/store/HomeStoreMapScreen/index.tsx` | 동일 |

#### single-page-ro — 수정 (2개)

| 파일 | 변경 내용 |
|------|-----------|
| `packages/map/src/components/map/util/send.ts` | postMessage origin "*" → 환경변수 기반 |
| `packages/map/src/components/map/hooks/useOnMessage.ts` | 수신 메시지 origin 검증 추가 |

### 총 변경: 신규 3개 + 수정 8개 = 11개 파일

---

### 후속 작업 (TODO)

- [ ] `MY_POSITION_PUBLIC_KEY` 환경변수에 Ed의 PEM 공개키 등록
- [ ] `GEOHASH_ENCRYPTION_KEY` 환경변수 등록 (BE와 동일 키)
- [ ] `NEXT_PUBLIC_PARENT_ORIGIN` 환경변수 설정 (single-page)
- [ ] B2C-49636 BE 배포 후 `sendEncryptedLocation.ts` API 엔드포인트 확정
- [ ] `common-zigbang-api-sdk` 메서드로 직접 API 호출 교체 검토
- [ ] 프리뷰 환경 테스트 (Ed 일정 조율)
- [ ] 프로덕션 배포 (BE Phase 1 완료 후)
