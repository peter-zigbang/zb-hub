# B2C-49770 FE Spec — zigbang-client + single-page

## 1. zigbang-client: Geohash 암호화

### 1.1 신규 파일

#### `packages/screens/src/lib/crypto/geohashCrypto.ts`

| 항목 | 내용 |
|------|------|
| 라이브러리 | `crypto-es` (기존 의존성) |
| 알고리즘 | AES-256-CBC |
| IV 생성 | HMAC-SHA256(plaintext, key).slice(0, 16) |
| 키 소스 | `process.env.GEOHASH_ENCRYPTION_KEY` (base64) |
| 출력 | `base64(IV + ciphertext)` |
| 캐싱 | 키 WordArray 1회 파싱 후 재사용 |

```typescript
// 핵심 API
export const encryptGeohash = (geohash: string): string
export const decryptGeohash = (encrypted: string): string
```

### 1.2 수정 파일

#### `packages/screens/src/lib/House/axios/index.ts`

request interceptor에 geohash 자동 암호화 추가:

```typescript
if (process.env.GEOHASH_ENCRYPTION_KEY) {
  if (config.params?.geohash) config.params.geohash = encryptGeohash(config.params.geohash)
  if (config.data?.geohash) config.data.geohash = encryptGeohash(config.data.geohash)
}
```

**암호화 대상 5개 화면 (interceptor로 일괄 처리):**

| 화면 | ScreenModel 위치 | geohash 전달 방식 |
|------|---|---|
| 원룸 | `HomeOneroomMapScreen/ScreenModel.ts` L399 | `makeOneroomFilterQuery(filter, geohash)` → params.geohash |
| 오피스텔 | `HomeOfficetelMapScreen/ScreenModel.ts` L400 | `makeRequestParams(geohash)` → params.geohash |
| 빌라 | `HomeVillaMapScreen/ScreenModel.ts` | `makeVillaFilterQueryV2(filter, geohash)` → params.geohash |
| 아파트 | `HomeAptMapScreen/ScreenModel.ts` L922,1055,1174,1308 | `getGeohashesByLocalZoomLevel()` → params.geohash |
| 상가 | `HomeStoreMapScreen/ScreenModel.ts` L387 | `query.geohash = geohash` → data.geohash |

---

## 2. zigbang-client: "내 위치찾기" RSA 암호화

### 2.1 신규 파일

#### `packages/screens/src/lib/crypto/locationCrypto.ts`

| 항목 | 내용 |
|------|------|
| API | Web Crypto API (`crypto.subtle`) |
| 알고리즘 | RSA-OAEP, SHA-256 |
| 키 소스 | `process.env.MY_POSITION_PUBLIC_KEY` (PEM) |
| 입력 | `JSON.stringify({ lat, lng })` |
| 출력 | base64 문자열 (~342자) |
| 캐싱 | CryptoKey 1회 import 후 재사용 |

```typescript
export const encryptLocation = async (lat: number, lng: number): Promise<string>
```

#### `packages/screens/src/lib/crypto/sendEncryptedLocation.ts`

| 항목 | 내용 |
|------|------|
| 엔드포인트 | `POST ${APIS_HOST}/v2/my-position` |
| 요청 바디 | `{ encryptedLocation: string }` |
| 에러 처리 | catch + console.warn (기존 기능 영향 없음) |
| 조건 | `MY_POSITION_PUBLIC_KEY` 미설정 시 skip |

### 2.2 수정 파일 (5개 화면 onUpdateLocation)

| 화면 | 파일 | 변경 내용 |
|------|------|-----------|
| 원룸 | `HomeOneroomMapScreen/index.tsx` | import + `sendEncryptedLocation(lat, lng)` 추가 |
| 오피스텔 | `HomeOfficetelMapScreen/index.tsx` | 동일 |
| 빌라 | `HomeVillaMapScreen/index.tsx` | 동일 |
| 아파트 | `HomeAptMapScreen/index.tsx` | 동일 |
| 상가 | `HomeStoreMapScreen/index.tsx` | 동일 |

**호출 위치:** `onUpdateLocation` 콜백 내, `animateToRegion()` 직전

---

## 3. single-page: postMessage 보안

### 3.1 수정 파일

#### `packages/map/src/components/map/util/send.ts`

| 변경 전 | 변경 후 |
|---------|---------|
| `window.parent.postMessage(payload, "*")` | `window.parent.postMessage(payload, MAP_PARENT_ORIGIN)` |

`MAP_PARENT_ORIGIN` = `process.env.NEXT_PUBLIC_PARENT_ORIGIN || "*"` (하위호환)

#### `packages/map/src/components/map/hooks/useOnMessage.ts`

수신 메시지 origin 검증 추가:
```typescript
const allowedOrigin = process.env.NEXT_PUBLIC_PARENT_ORIGIN
if (allowedOrigin && event.origin !== allowedOrigin) return
```

---

## 4. 환경변수 요약

| 변수 | 레포 | 값 | 필수 |
|------|------|---|------|
| `GEOHASH_ENCRYPTION_KEY` | zigbang-client | base64 인코딩 32바이트 | 암호화 활성화 시 |
| `MY_POSITION_PUBLIC_KEY` | zigbang-client | PEM 공개키 문자열 | 위치 전송 활성화 시 |
| `NEXT_PUBLIC_PARENT_ORIGIN` | single-page | e.g. `https://www.zigbang.com` | 권장 |

---

## 5. 의존성

| 패키지 | 버전 | 용도 | 상태 |
|--------|------|------|------|
| `crypto-es` | ^1.2.6 | AES-256-CBC | 기존 설치됨 |
| Web Crypto API | 내장 | RSA-OAEP | 브라우저/RN 내장 |

신규 의존성 추가 없음.

---

## 6. 참고: BE 연동 (B2C-49636)

"내 위치보기" API 엔드포인트(`POST /v2/my-position`)는 B2C-49636 배포 후 확정 필요.
현재 `sendEncryptedLocation.ts`의 경로는 임시값이며, SDK(`common-zigbang-api-sdk`) 메서드로 교체 예정.

API 문서: https://apis.zigbang.net/common/api/#/Index/IndexController_saveLocation
