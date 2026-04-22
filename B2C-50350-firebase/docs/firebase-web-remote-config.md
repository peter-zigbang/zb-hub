# Firebase Remote Config 웹 지원 작업 내역

## 작업 목적

기존 프로젝트에서 Firebase Remote Config는 **React Native (iOS/Android)** 에서만 동작했고, 웹(`zigbang-m`, `zigbang-www`)에서는 `@zigbang/bridge/firebase` 스텁이 로컬 defaults만 반환하는 상태였음. 웹에서도 Firebase 서버의 Remote Config 값을 실시간으로 가져오도록 구현.

---

## 변경 사항 요약

| 영역 | 파일 | 변경 내용 |
|------|------|----------|
| 의존성 | `packages/bridge/package.json` | `firebase: ^11.0.0` 추가 |
| 구현 | `packages/bridge/src/lib/firebase/index.ts` | Firebase JS SDK 기반 Remote Config 구현 + `initApp(config)` 메서드 추가 |
| 초기화 | `packages/zigbang-m/pages/_app.tsx` | `Firebase.initApp(...)` 호출 추가 |
| 초기화 | `packages/zigbang-www/pages/_app.tsx` | `Firebase.initApp(...)` 호출 추가 |
| 환경변수 | `packages/zigbang-m/.env` + `ci/.env.{dev,preview,prod,release}` | Firebase 웹 config 6종 추가 |
| 환경변수 | `packages/zigbang-www/.env` + `ci/.env.{dev,preview,prod,release}` | Firebase 웹 config 6종 추가 |
| 테스트 | `packages/screens/src/home/GatewayScreenAB.m.tsx` | Remote Config fetch 테스트 로그 추가 (검증 후 제거 예정) |

---

## 1. `@zigbang/bridge/firebase` 웹 구현

### 이전 (스텁)

```typescript
static async fetchAndActivate() {
  return false  // 아무 동작 안 함
}
static getValue(key: string) {
  return removeConfigDefault[key]?.toString()  // 로컬 defaults만
}
```

### 이후 (Firebase JS SDK 사용)

```typescript
import { getApps, initializeApp, type FirebaseOptions } from "firebase/app"
import {
  fetchAndActivate,
  getRemoteConfig,
  getValue as getRemoteConfigValue,
  // ...
} from "firebase/remote-config"

export class Firebase {
  // 웹에서만 사용. 웹앱 entry에서 1회 호출
  static initApp(config: FirebaseOptions) { ... }

  static remoteConfig = class {
    static async fetchAndActivate(expirationDuration?: number) {
      const rc = getRC()
      // 실제 Firebase 서버에서 fetch + activate
      return await fetchAndActivate(rc)
    }
    static getValue(key: string): string | undefined {
      const rc = getRC()
      return getRemoteConfigValue(rc, key).asString() || undefined
    }
    // ...
  }
}
```

### 디버깅 로그 (🟡 이모지)

- `🟡 Firebase: initApp success/failed` → 초기화 상태
- `🟡 RC: fetchAndActivate 시작/결과` → fetch 성공 여부
- `🟡 RC: lastFetchStatus` → Firebase SDK 내부 상태 (success/failure/throttle)
- `🟡 RC: getValue { key, source, value }` → 어떤 key가 어디서(remote/default/static) 왔는지

---

## 2. 웹앱 Firebase 초기화

### `packages/zigbang-m/pages/_app.tsx` / `packages/zigbang-www/pages/_app.tsx`

```typescript
import { Firebase } from "@zigbang/bridge/firebase"

Firebase.initApp({
  apiKey: process.env.NEXT_PUBLIC_FIREBASE_API_KEY ?? "",
  authDomain: process.env.NEXT_PUBLIC_FIREBASE_AUTH_DOMAIN ?? "",
  projectId: process.env.NEXT_PUBLIC_FIREBASE_PROJECT_ID ?? "",
  storageBucket: process.env.NEXT_PUBLIC_FIREBASE_STORAGE_BUCKET ?? "",
  messagingSenderId: process.env.NEXT_PUBLIC_FIREBASE_MESSAGING_SENDER_ID ?? "",
  appId: process.env.NEXT_PUBLIC_FIREBASE_APP_ID ?? "",
})
```

`_app.tsx` 모듈 로드 시점에 1회 호출. 이후 `Firebase.remoteConfig.*` 메서드가 이 app 인스턴스를 사용.

---

## 3. 환경별 Firebase 웹앱 등록

Firebase Console에서 **3개 웹앱** 신규 등록:

| 환경 | 앱 닉네임 | appId |
|------|----------|-------|
| dev | 직방 WWW dev | `1:<SENDER_ID>:web:<APP_ID_DEV>` |
| preview | 직방 WWW preview | `1:<SENDER_ID>:web:<APP_ID_PREVIEW>` |
| prod | 직방 WWW | `1:<SENDER_ID>:web:<APP_ID_PROD>` |

공통 설정 (실제 값은 Firebase Console / 사내 시크릿 스토어에서 확인):
- Project ID: `<FIREBASE_PROJECT_ID>`
- API Key: `<FIREBASE_WEB_API_KEY>`
- Auth Domain: `<FIREBASE_PROJECT_ID>.firebaseapp.com`
- Storage Bucket: `<FIREBASE_PROJECT_ID>.appspot.com`
- Messaging Sender ID: `<SENDER_ID>`

### env 파일 매핑

| 파일 | appId |
|------|-------|
| `.env` (로컬) | dev |
| `ci/.env.dev` | dev |
| `ci/.env.preview` | preview |
| `ci/.env.prod` | prod |
| `ci/.env.release` | prod |

`zigbang-m`, `zigbang-www` 양쪽 모두 동일 매핑. 총 10개 env 파일 업데이트.

---

## 4. 검증 결과

### 테스트 파라미터

Firebase Console → Remote Config:
- Key: `firebase_web_enable_test`
- 기본값: `hello_zigbang` (string)

### 테스트 코드 (`GatewayScreenAB.m.tsx`)

```typescript
React.useEffect(() => {
  const loadRemoteConfig = async () => {
    console.log("🟡 RC: fetchAndActivate 시작")
    const activated = await Firebase.remoteConfig.fetchAndActivate()
    console.log("🟡 RC: fetchAndActivate 결과", activated)
    const value = Firebase.remoteConfig.getValue("firebase_web_enable_test")
    console.log("🟡 RC: firebase_web_enable_test =", value)
  }
  loadRemoteConfig()
}, [])
```

### 실제 콘솔 출력 (성공)

```
🟡 Firebase: initApp success { projectId: '<FIREBASE_PROJECT_ID>', appId: '1:<SENDER_ID>:web:<APP_ID_DEV>' }
🟡 RC: fetchAndActivate 시작
🟡 RC: settings { minimumFetchIntervalMillis: 43200000, fetchTimeoutMillis: 60000 }
🟡 RC: lastFetchStatus success
🟡 RC: fetchAndActivate 결과 true
🟡 RC: getValue { key: 'firebase_web_enable_test', source: 'remote', value: 'hello_zigbang' }
🟡 RC: firebase_web_enable_test = hello_zigbang
```

`source: 'remote'` → Firebase 서버에서 실시간으로 가져온 값. 정상 동작 확인.

---

## 5. 트러블슈팅 이력

### 이슈 1: `appId: ''` → `getRemoteConfig` 실패

**증상**
```
🟡 Firebase: initApp success { projectId: '...', appId: '' }
🟡 RC: getRemoteConfig 실패 (Firebase app 미초기화)
```

**원인**: `NEXT_PUBLIC_FIREBASE_APP_ID` env 미설정. Firebase Web SDK는 `appId` 없으면 Remote Config 서버 인증 불가.

**해결**: Firebase Console에서 웹앱 등록 후 `appId` 획득 → `.env`에 입력.

### 이슈 2: 12시간 캐시

Firebase Remote Config 기본 `minimumFetchIntervalMillis = 43200000` (12시간). 같은 세션에서 콘솔 값 변경 후 즉시 반영이 안 되면 `reset()` 후 `fetchAndActivate()` 호출 필요.

---

## 6. 기존 사용처 (자동 적용됨)

기존에 `Firebase.remoteConfig.*`를 호출하던 코드들이 **수정 없이** 웹에서도 동작하게 됨:

- `packages/screens/src/WebAppView/Component.tsx` — `getValue(key)`
- `packages/screens/src/home/stay/StayWebAppView/Component.tsx` — `getValue(key)`
- `packages/screens/src/login/LoginStartScreen.tsx` — `reset()`, `fetchAndActivate()`

---

## 7. 후속 작업

### 필수
- [ ] `yarn install` 실행하여 `firebase` 패키지 설치
- [ ] `GatewayScreenAB.m.tsx` 테스트 코드 제거 (검증 완료 후)
- [ ] `bridge/src/lib/firebase/index.ts`의 상세 디버그 로그 정리 (운영 시 `getValue` 로그는 호출마다 찍힘)

### 권장
- [ ] Firebase Console → API Key 제한에서 **도메인 제한** 설정
  - `m.zigbang.com`, `www.zigbang.com` → prod
  - preview/dev 도메인별로 분리
- [ ] Remote Config 기본값(`packages/screens/src/lib/Firebase/remoteConfigDefaults.json`)을 웹 초기화 시에도 `Firebase.remoteConfig.initialize()` 호출로 주입하도록 연결
- [ ] 웹에서 사용할 추가 Remote Config 파라미터 정의 및 타입 안전 래퍼 작성

---

## 참고 자료

- Firebase Web SDK: https://firebase.google.com/docs/web/setup
- Remote Config (Web): https://firebase.google.com/docs/remote-config/get-started?platform=web
- 프로젝트 ID: `<FIREBASE_PROJECT_ID>` (실제 값은 Firebase Console 참조)
