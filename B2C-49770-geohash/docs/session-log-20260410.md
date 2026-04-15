# [B2C-49770] locationCrypto PEM 파싱 및 MGF1 해시 수정

- **커밋**: `a2d200d719`
- **파일**: `packages/screens/src/lib/crypto/locationCrypto.ts`

### 변경 내용

#### 1. PEM 환경변수 `\n` 이스케이프 → 실제 줄바꿈 변환

```diff
- cachedPublicKey = forge.pki.publicKeyFromPem(MY_POSITION_PUBLIC_KEY)
+ cachedPublicKey = forge.pki.publicKeyFromPem(MY_POSITION_PUBLIC_KEY.replace(/\\n/g, "\n"))
```

- CI/CD 환경변수에 PEM 키를 저장할 때 줄바꿈이 `\n` 문자열 리터럴로 들어옴
- `forge.pki.publicKeyFromPem()`은 실제 줄바꿈이 필요하므로 파싱 실패
- `.replace(/\\n/g, "\n")`으로 변환하여 해결

#### 2. RSA-OAEP MGF1 해시를 SHA-256으로 명시 (BE KMS 호환)

```diff
  const encrypted = publicKey.encrypt(plaintext, "RSA-OAEP", {
    md: forge.md.sha256.create(),
+   mgf1: {
+     md: forge.md.sha256.create(),
+   },
  })
```

- node-forge의 RSA-OAEP 기본 MGF1 해시는 SHA-1
- BE(AWS KMS)는 OAEP 해시와 MGF1 해시 모두 SHA-256을 사용
- MGF1 해시를 명시하지 않으면 BE에서 복호화 실패
