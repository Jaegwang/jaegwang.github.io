--- 
title: "크래시 리포트를 읽을 수 있게 만드는 디버그 심볼의 모든 것"
date: 2025-12-24 00:00:00 +0900
categories: [DevOps, Sentry]
tags: [Debug, Symbol, CI/CD, Sentry, Crash Report]
---

> 크래시 리포트를 읽을 수 있게 만드는 디버그 심볼의 모든 것

## 목차

1. [디버그 심볼이란?](#1-디버그-심볼이란)
2. [왜 디버그 심볼이 필요한가?](#2-왜-디버그-심볼이-필요한가)
3. [빌드마다 업로드가 필요한 이유](#3-빌드마다-업로드가-필요한-이유)
4. [플랫폼별 업로드 방법](#4-플랫폼별-업로드-방법)
5. [Debug ID: 심볼 매칭의 비밀](#5-debug-id-심볼-매칭의-비밀)
6. [Debug ID 확인 방법](#6-debug-id-확인-방법)
7. [심볼리케이션: 실제 동작 원리](#7-심볼리케이션-실제-동작-원리)
8. [CI/CD 자동화](#8-cicd-자동화)
9. [자주 묻는 질문](#9-자주-묻는-질문)

---

## 1. 디버그 심볼이란?

디버그 심볼은 **컴파일된 코드와 원본 소스코드를 연결해주는 매핑 정보**입니다.

프로그램을 빌드하면 소스코드가 기계어로 변환되면서 함수명, 변수명, 파일 위치 등의 정보가 사라집니다. 디버그 심볼은 이 정보를 별도 파일에 보관하여, 나중에 크래시가 발생했을 때 "어디서 문제가 생겼는지" 알 수 있게 해줍니다.

### 플랫폼별 디버그 심볼 파일

| 플랫폼 | 파일 형식 | 설명 |
|--------|----------|------|
| Windows | `.pdb` | Program Database |
| iOS/macOS | `.dSYM` | Debug Symbol 번들 |
| Android (Java) | `mapping.txt` | ProGuard/R8 난독화 매핑 |
| Android (NDK) | `.so` (with debug) | 네이티브 라이브러리 |
| JavaScript | `.map` | 소스맵 |

---

## 2. 왜 디버그 심볼이 필요한가?

### 디버그 심볼이 없을 때

```
크래시 리포트:
Exception: ACCESS_VIOLATION

Stack Trace:
  #0  0x00007FF6A1B23456  MyApp.exe + 0x23456
  #1  0x00007FF6A1B21234  MyApp.exe + 0x21234
  #2  0x00007FF6A1B20100  MyApp.exe + 0x20100

→ 메모리 주소만 보여서 어디서 크래시났는지 알 수 없음 😢
```

### 디버그 심볼이 있을 때

```
크래시 리포트:
Exception: ACCESS_VIOLATION (Null pointer dereference)

Stack Trace:
  #0  PlayerController::TakeDamage(int damage)
      at src/PlayerController.cpp:142
      
  #1  GameEngine::Update()
      at src/GameEngine.cpp:89
      
  #2  main()
      at src/main.cpp:25

→ 정확한 함수명과 소스코드 위치를 알 수 있음! 🎉
```

---

## 3. 빌드마다 업로드가 필요한 이유

**핵심: 빌드할 때마다 코드의 메모리 배치가 달라집니다.**

같은 소스코드라도 빌드할 때마다:
- 함수들의 메모리 주소가 달라짐
- 변수명 매핑이 달라짐 (minify/obfuscate)
- 최적화 수준에 따라 구조가 달라짐

따라서 **프로덕션에 배포되는 모든 고유한 빌드**에 대해 해당 디버그 심볼을 업로드해야 합니다.

```
빌드 v1.0 (build #100)  →  심볼 파일 A  →  Sentry에 업로드
빌드 v1.0 (build #101)  →  심볼 파일 B  →  Sentry에 업로드 (다른 파일!)
빌드 v1.1 (build #102)  →  심볼 파일 C  →  Sentry에 업로드
```

---

## 4. 플랫폼별 업로드 방법

### 네이티브 (iOS dSYM, Android NDK, Windows PDB)

```bash
sentry-cli debug-files upload \
  --org <조직명> \
  --project <프로젝트명> \
  <심볼 파일 경로>
```

**예시:**
```bash
# iOS dSYM
sentry-cli debug-files upload --org my-org --project my-app ./MyApp.dSYM

# Android .so 파일
sentry-cli debug-files upload --org my-org --project my-app ./app/build/intermediates/ndkBuild/

# Windows PDB
sentry-cli debug-files upload --org my-org --project my-app ./build/Release/
```

### JavaScript 소스맵

```bash
sentry-cli sourcemaps upload \
  --org <조직명> \
  --project <프로젝트명> \
  --release <버전> \
  ./dist
```

**예시:**
```bash
sentry-cli sourcemaps upload \
  --org my-org \
  --project my-web-app \
  --release "my-app@1.2.3" \
  ./dist
```

### Android ProGuard/R8

```bash
sentry-cli upload-proguard \
  --org <조직명> \
  --project <프로젝트명> \
  ./app/build/outputs/mapping/release/mapping.txt
```

---

## 5. Debug ID: 심볼 매칭의 비밀

**"Sentry는 수많은 심볼 파일 중에서 어떻게 올바른 것을 찾을까?"**

답은 **Debug ID**입니다.

### 네이티브 빌드의 경우

바이너리 파일과 심볼 파일에 **동일한 고유 ID(UUID/Build ID)**가 컴파일 시점에 자동으로 내장됩니다.

```
┌─────────────────────────────────────────────────────┐
│ 크래시 리포트                                         │
│ Debug ID: 3E3C6E2B-1A2B-3C4D-5E6F-7A8B9C0D1E2F     │
│ 크래시 주소: 0x00012345                              │
└─────────────────────────────────────────────────────┘
                        │
                        ▼ Sentry가 Debug ID로 검색
┌─────────────────────────────────────────────────────┐
│ 업로드된 심볼 저장소                                   │
│                                                     │
│ ├─ Debug ID: 3E3C6E2B-... → MyApp_v1.0 ✅ 매칭!     │
│ ├─ Debug ID: 8F9A0B1C-... → MyApp_v1.1             │
│ └─ Debug ID: 2D3E4F5A-... → MyApp_v1.2             │
└─────────────────────────────────────────────────────┘
```

### JavaScript의 경우

소스맵은 자동 Debug ID가 없어서 **release + dist** 조합으로 매칭합니다.

```javascript
// SDK 초기화 시
Sentry.init({
  dsn: "...",
  release: "my-app@1.2.3",  // 버전
  dist: "build-456"         // 빌드 번호
});
```

```bash
# 업로드 시 동일하게 지정
sentry-cli sourcemaps upload \
  --release "my-app@1.2.3" \
  --dist "build-456" \
  ./dist
```

### 매칭 방식 요약

| 플랫폼 | 매칭 방식 | 자동/수동 |
|--------|----------|----------|
| iOS (dSYM) | Debug ID (UUID) | ✅ 자동 생성 |
| Android NDK | Debug ID (Build ID) | ✅ 자동 생성 |
| Windows (PDB) | Debug ID (GUID) | ✅ 자동 생성 |
| JavaScript | release + dist | ⚠️ 수동 지정 |
| ProGuard | release + UUID | 반자동 |

---

## 6. Debug ID 확인 방법

### sentry-cli 사용 (모든 플랫폼)

```bash
sentry-cli debug-files check <파일 경로>

# 예시
sentry-cli debug-files check ./MyApp.dSYM
sentry-cli debug-files check ./MyApp.exe
sentry-cli debug-files check ./libnative.so
```

**출력 예시:**
```
Debug Info File Check
  Type: dsym
  Identifier: 3e3c6e2b-1a2b-3c4d-5e6f-7a8b9c0d1e2f
  Arch: arm64
```

### iOS/macOS (dwarfdump)

```bash
# dSYM 파일
dw arfdump --uuid ./MyApp.app.dSYM

# 바이너리 파일
dw arfdump --uuid ./MyApp.app/MyApp
```

**출력:**
```
UUID: 3E3C6E2B-1A2B-3C4D-5E6F-7A8B9C0D1E2F (arm64)
```

### Android NDK (readelf)

```bash
readelf -n ./libnative.so | grep "Build ID"
```

**출력:**
```
Build ID: 3e3c6e2b1a2b3c4d5e6f7a8b9c0d1e2f
```

### Sentry에 업로드된 심볼 목록 확인

```bash
sentry-cli debug-files list --org <조직명> --project <프로젝트명>
```

---

## 7. 심볼리케이션: 실제 동작 원리

크래시가 발생하면 어떤 과정을 거쳐 읽을 수 있는 스택 트레이스가 되는지 알아봅시다.

### Step 1: 크래시 캡처 (클라이언트)

프로그램이 크래시되면 **메모리 주소**만 캡처됩니다.

```
원시 크래시 데이터:
- Debug ID: 3E3C6E2B-...
- 크래시 주소: 0x23456, 0x21234, 0x20100
```

### Step 2: PDB/dSYM 파일의 내용

디버그 심볼 파일에는 **주소 ↔ 소스코드 매핑 테이블**이 있습니다.

```
심볼 파일 내부:

[Symbol Table - 주소 → 함수명]
0x23400 - 0x23500  →  PlayerController::TakeDamage()
0x21200 - 0x21300  →  GameEngine::Update()
0x20100 - 0x20200  →  main()

[Line Table - 주소 → 파일:라인]
0x23456  →  src/PlayerController.cpp:142
0x21234  →  src/GameEngine.cpp:89
0x20100  →  src/main.cpp:25
```

### Step 3: 심볼리케이션 (Sentry 서버)

Sentry가 Debug ID로 맞는 심볼 파일을 찾아서 주소를 변환합니다.

```
변환 과정:
0x23456  →  PlayerController::TakeDamage() at PlayerController.cpp:142
0x21234  →  GameEngine::Update() at GameEngine.cpp:89
0x20100  →  main() at main.cpp:25
```

### Step 4: 최종 결과

Sentry UI에서 읽을 수 있는 스택 트레이스를 확인할 수 있습니다!

### 전체 흐름 다이어그램

```
┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│    빌드 시점     │     │   런타임 (유저)  │     │   Sentry 서버   │
├─────────────────┤     ├─────────────────┤     ├─────────────────┤
│                 │     │                 │     │                 │
│  소스코드        │     │  앱 실행        │     │  Debug ID로     │
│     ↓           │     │     ↓           │     │  심볼 파일 검색  │
│  컴파일         │     │  크래시 발생!   │     │     ↓           │
│     ↓           │     │     ↓           │     │  주소 → 함수명  │
│  .exe + .pdb    │     │  메모리 주소    │     │  주소 → 파일:줄 │
│     ↓           │     │  + Debug ID    │     │     ↓           │
│  심볼을 Sentry  │────→│     ↓           │────→│  읽을 수 있는   │
│  에 업로드      │     │  Sentry로 전송  │     │  스택 트레이스!  │
│                 │     │                 │     │                 │
└─────────────────┘     └─────────────────┘     └─────────────────┘
```

---

## 8. CI/CD 자동화

매 빌드마다 수동으로 업로드하는 것은 번거롭습니다. CI/CD에 통합하세요!

### GitHub Actions 예시

```yaml
name: Build and Upload Symbols

on:
  push:
    branches: [main]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Build
        run: npm run build
        
      - name: Upload source maps to Sentry
        env:
          SENTRY_AUTH_TOKEN: ${{ secrets.SENTRY_AUTH_TOKEN }}
          SENTRY_ORG: my-org
          SENTRY_PROJECT: my-project
        run: |
          npm install -g @sentry/cli
          sentry-cli sourcemaps upload \
            --release "${{ github.sha }}" \
            ./dist
```

### Gradle (Android) 자동화

```groovy
// build.gradle
plugins {
    id "io.sentry.android.gradle" version "3.x.x"
}

sentry {
    uploadNativeSymbols = true
    autoUploadProguardMapping = true
    org = "my-org"
    projectName = "my-android-app"
}
```

### Xcode (iOS) 자동화

Build Phases에 Run Script 추가:

```bash
if which sentry-cli >/dev/null; then
  export SENTRY_ORG=my-org
  export SENTRY_PROJECT=my-ios-app
  sentry-cli debug-files upload --include-sources "$DWARF_DSYM_FOLDER_PATH"
else
  echo "warning: sentry-cli not installed"
fi
```

---

## 9. 자주 묻는 질문

### Q: 모든 빌드에 심볼을 업로드해야 하나요?

**A:** 프로덕션에 배포되는 빌드만 업로드하면 됩니다. 로컬 개발/디버그 빌드는 보통 불필요합니다.

### Q: 심볼 파일이 없으면 어떻게 되나요?

**A:** 크래시는 수집되지만, 스택 트레이스에 메모리 주소만 표시되어 디버깅이 매우 어렵습니다.

### Q: 심볼 파일은 얼마나 보관되나요?

**A:** Sentry 플랜에 따라 다릅니다. 일반적으로 릴리스와 함께 관리되며, 오래된 릴리스의 심볼은 정리할 수 있습니다.

### Q: Debug ID가 일치하지 않으면?

**A:** 심볼리케이션이 실패하고 "Missing debug files" 경고가 표시됩니다. 올바른 빌드의 심볼을 업로드했는지 확인하세요.

### Q: 소스코드도 업로드되나요?

**A:** 기본적으로 소스코드는 업로드되지 않습니다. `--include-sources` 옵션을 사용하면 소스 컨텍스트를 포함할 수 있습니다.

---

## 마무리

디버그 심볼 관리는 처음에는 복잡해 보이지만, CI/CD에 한번 설정해두면 이후로는 자동으로 처리됩니다.

**핵심 요약:**
1. 프로덕션 빌드마다 심볼 업로드 필요
2. 네이티브는 Debug ID로 자동 매칭
3. JavaScript는 release + dist로 수동 매칭
4. CI/CD 자동화로 편하게 관리

이제 Sentry에서 크래시가 발생해도 정확한 위치를 바로 확인할 수 있습니다! 🚀
