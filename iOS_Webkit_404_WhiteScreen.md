---
tags:
  - iOS
  - WebKit
  - VueJS
  - Troubleshooting
  - Cross-Platform
date: 2025-12-04
---
🟢 [iOS/WebKit] 리소스 404 에러가 SPA 앱 크래시(White Screen)으로 **전파(Propagate)되는 메커니즘** 분석

> **Date:** 2025.12.04  
> **Tags:** `#iOS` `#WebKit` `#Vue.js` `#Troubleshooting` `#Cross-Platform`  
> **Environment:** iOS Safari (Random), Vue Router v4.x

---

## 1. 🚨 증상 (Symptoms)

- **현상:** 특정 iOS 단말 및 앱 버전에서 페이지 진입 시 간헐적으로 화면이 하얗게 멈추는 **White Screen (Crash)** 현상 발생.
- **특이사항:**
    - 동일한 코드임에도 **Android(Chrome/Blink) 및 데스크탑 환경에서는 정상 작동** (이미지만 엑스박스로 뜨고 앱은 구동됨).
    - 오직 **iOS WebKit 환경**에서만 치명적인 오류로 전파됨.
- **Error Log:**
    ```javascript
    Unhandled Promise Rejection: Error
    (anonymous function) - chunk-vendors.js (vue-router)
    ```

---

## 2. 🕵️‍♂️ 원인 분석 (Root Cause Analysis)

### 2.1. 1차 원인: 잘못된 리소스 경로
- 인라인 스타일(`style`) 내 `background-image` 경로에 Webpack Alias(`@/assets/...`)가 빌드되지 않은 채 그대로 포함됨.
- 브라우저가 실제 URL로 `@` 문자가 포함된 주소를 요청하여 **HTTP 404 Error** 발생.

### 2.2. 심층 분석: 왜 iOS에서만 앱이 죽었는가? (WebKit vs Blink)
단순한 이미지 로딩 실패가 앱 전체 크래시로 이어진 원인을 분석함.

1.  **Vue Router의 동작 원리:**
    - Vue Router의 네비게이션 가드(`beforeEnter` 등)와 컴포넌트 마운트는 하나의 **Promise Chain**으로 연결되어 비동기적으로 처리됨.

2.  **Blink (Chrome) 엔진:**
    - 리소스 로딩 에러(404)를 **Non-blocking**으로 처리.
    - 렌더링 스레드와 JS 실행 스레드가 분리되어 있어, 이미지가 안 떠도 JS 로직(라우팅)은 계속 진행됨.
    
3.  **WebKit (Safari) 엔진의 특수성 (Race Condition):**
    - 페이지 트랜지션 중 발생하는 리소스 에러를 **현재 실행 중인 Promise 컨텍스트의 치명적 실패(Rejection)**로 간주하는 경향이 있음.
    - **가설:** 컴포넌트 마운트 시점에 발생한 404 에러가 Vue Router의 비동기 네비게이션 Promise에 `throw` 되었고, 이것이 Catch 되지 않아 **네비게이션 프로세스 자체가 중단(Abort)**된 것으로 판단됨.
    
4.  **추정되는 에러 발생 시나리오 (The Crash Moment):**
    - 문제는 **이전 페이지가 `Unmount` 되고, 새로운 페이지의 비동기 컴포넌트(Chunk)가 로딩되거나 최초 렌더링을 시도하는 단계** 사이에서 발생한 것으로 추정됨.
    
    1.  라우터가 이동을 시작하고, **비동기 컴포넌트(새 페이지)**를 로딩함.
    2.  이 과정에서 WebKit이 인라인 스타일 내의 이미지 경로(404)를 확인하고 에러를 발생시킴.
    3.  WebKit은 이것을 단순 "리소스 로딩 실패"가 아니라 **"비동기 컴포넌트 로딩 실패(Promise Rejection)로 처리하여 상위 컨텍스트로 던짐.
    4.  Vue Router는 **"새 페이지 로딩 실패"로 인식하여 네비게이션 프로세스를 중단(Abort)함.
    5.  이미 뷰포트는 갱신 단계에 진입하여 이전 화면을 지웠으므로, **이전 화면도 없고 새 화면도 없는 'White Screen' 상태**에 빠짐.

---

## 3. 🛠️ 해결 및 조치 (Solution)

### 3.1. Hotfix (즉시 조치)
- 문제가 된 인라인 스타일 내 이미지 경로 수정.
- HTTP 404 에러를 제거하여 크래시 원천 차단.

### 3.2. Refactoring (재발 방지 / 아키텍처 개선)
**"인라인 스타일은 빌드 타임에 검증되지 않는다"**는 구조적 위험을 제거하기 위해 스타일링 방식 변경.

- **AS-IS:** Inline Style (String Interpolation)
    ```html
    <div style="background-image: url('@/assets/img.png')"></div>
    ```
- **TO-BE:** CSS Class / Tailwind (Compile Time Check)
    ```css
    /* CSS 로더가 빌드 타임에 '@' Alias를 해석하여 실제 경로로 변환함 */
    .bg-custom { background-image: url('~@/assets/img.png'); }
    ```
- **효과:** 경로 오타 발생 시 **컴파일 단계에서 에러(Build Fail)**를 뱉으므로, 런타임에서 사용자가 404로 인한 크래시를 겪을 확률 **0%**로 차단.

---

## 4. 💡 인사이트 (Key Takeaways)

1.  **크로스 브라우징의 깊이:** 단순히 CSS가 다르게 보이는 수준을 넘어, **에러를 핸들링하는 엔진 레벨의 정책(Engine Policy)**이 다를 수 있음을 확인함.
2.  **방어적 코딩:** 모바일 하이브리드 앱 환경에서는 작은 리소스 에러도 치명적인 UX 결함으로 이어질 수 있으므로, **엄격한 린트(Lint) 규칙과 빌드 타임 검증**이 필수적임.