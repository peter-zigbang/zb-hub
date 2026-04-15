# B2C-49770 코드 분석: zigbang-client

## Geohash 사용처 (5개 맵 화면)

### 1. 원룸 (Oneroom)
- `packages/screens/src/home/oneroom/HomeOneroomMapScreen/ScreenModel.ts:144, 210-230`
- `packages/screens/src/lib/House/oneroom/list.ts:47-65`
  - `makeRequestParams()` → `makeOneroomFilterQuery(filter, geohash)`
  - `Geohash.encode(latitude, longitude, vipSetting.geohashLength)`

### 2. 오피스텔 (Officetel)
- `packages/screens/src/home/officetel/HomeOfficetelMapScreen/ScreenModel.ts:398-421`
- `packages/screens/src/lib/House/officetel/list.ts:119-128`
  - `makeRequestParamsV2()` → `makeOfficetelFilterQuery(filter, geohash)`

### 3. 빌라 (Villa)
- `packages/screens/src/home/villa/HomeVillaMapScreen/ScreenModel.ts:643-786`

### 4. 아파트 (Apt)
- `packages/screens/src/home/apt/HomeAptMapScreen/ScreenModel.ts:1308-1950`
  - `getGeohashesByLocalZoomLevel()`로 zoom별 geohash 배치 생성

### 5. 상가 (Store)
- `packages/screens/src/home/store/HomeStoreMapScreen/ScreenModel.ts:378-410`
- `packages/screens/src/home/store/StoreAPI.ts:55-63`
  - `상가지도조회(params)` — `geohash?: string`

## Geohash 최적화 로직
- `packages/screens/src/lib/GeohashPolicy.ts`
  - `optimize()`: 인접 geohash 8개 → 부모로 병합 (API 호출 최소화)
  - `getLengthByZoom()`: zoom별 정밀도 결정 (4-5자)

## API 요청 흐름
```
Map onRegionChangeComplete()
  → Geohash.encode(lat, lng, length)
  → GeohashPolicy.optimize(geohashArray)
  → makeFilterQuery(filter, geohash)   ← 여기서 암호화 필요
  → itemApis.getItems(params)          ← axios로 전송
```

## 기존 암호화 유틸리티
- `packages/zigbang-www/lib/zbUserCrypto.ts`
  - AES-128-CBC (decrypt만) — 재사용 불가
- `@aws-crypto/sha256-js`: 의존성 존재
- `crypto-es`: devDependencies에 존재

## "내 위치찾기" 관련
- `packages/screens/src/home/maps/components/MapControllers.tsx:74, 114`
  - 버튼: `TestID("내위치")` → `onMyLocationPress()` 콜백
- `packages/screens/src/lib/GeolocationService/index.native.ts`
  - `getCurrentPosition()`: lat/lng 반환
  - `StorageManager` → `Constants.KEY_LOCATION_STORE`에 저장

## Axios 설정
- `packages/screens/src/lib/House/axios/index.ts`
  - Request Interceptor (line 33-49): 헤더 추가
  - Response Interceptor (line 17-31): 재시도 로직

## SDK 사용 패턴
```typescript
import { Item } from "@zigbang/house-api-sdk"
const itemApis = new Item({ basePath: process.env.APIS_HOST, axios: axiosInstance })
await itemApis.getOneroomItems(params)
```
- `common-zigbang-api-sdk-v0.4.73` — "내 위치찾기" 신규 API용

## 구현 전략 (제안)

### 작업 1: Geohash AES-256-CBC 암호화
- **암호화 위치:** `makeFilterQuery()` 호출 전 또는 axios request interceptor
- **수정 파일:**
  - 신규: `packages/screens/src/lib/crypto/geohashCrypto.ts`
  - 수정: `packages/screens/src/lib/House/axios/index.ts` (interceptor)
  - 또는 개별: `oneroom/list.ts`, `officetel/list.ts`, `StoreAPI.ts`

### 작업 2: "내 위치찾기" RSA-OAEP 암호화
- **암호화 위치:** `onMyLocationPress()` → API 호출 전
- **수정 파일:**
  - 신규: `packages/screens/src/lib/crypto/locationCrypto.ts`
  - 수정: 내 위치 버튼 핸들러에서 `saveLocation` API 호출 추가
- **의존성:** Web Crypto API (브라우저 내장) 또는 `jose` 라이브러리
