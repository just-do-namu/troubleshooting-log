---
tags:
  - Hybrid-App
  - iOS
  - Android
  - WKWebView
  - Custom-Scheme
  - Header-Spoofing
  - Troubleshooting
  - SDK-Analysis
date: 2025-12-10
---

🟢 [분석/Hybrid] 로컬 디버깅 환경에서의 3rd Party SDK 인증 실패 원인 및 OS별 동작 차이 분석

> **Date:** 2025.12.11
> **Tags:** #HybridApp #iOS #Android #CustomScheme #HeaderSpoofing #SDK_Internals

---

## 1. 🚨 현상 (Observation)

- **Debug Environment:** PC 로컬 웹서버를 기동하고, 모바일 디바이스에서 해당 PC의 IP(`http://192.168.x.x`)로 접속하여 디버깅 진행.

- **Issue:** **Naver Map SDK** 연동 시 OS별로 상이한 동작 발생.
    - **Android:** `http://192.168.x.x` 로 접속 시, 지도가 **0.5초간 정상 노출**된 후 "인증 실패" 메시지와 함께 화면이 차단(Dimmed)됨.
    
    - **iOS:** `http` 프로토콜 인터셉트 제약으로 인해 **`custom://localhost`** 스킴으로 우회 접속했으나, 오히려 **지속적으로 정상 작동**함.


## 2. 🕵️‍♂️ 원인 분석 (Root Cause Analysis)

### 2.1 Android 실패 원인: CORS가 아닌 SDK 내부 차단 로직

- **Not CORS:** ==지도가 0.5초간 정상적으로 보였다==는 점은, 지도 타일 이미지나 JS ==리소스 자체를 가져오는 데는 성공==했음을 의미함. 만약 **CORS(Network Level)** 문제였다면 리소스 로딩 단계에서 브라우저가 차단하여 화면이 아예 뜨지 않았을 것임.

- **Application Level Block:** 1.  SDK가 초기 리소스를 로딩하여 지도를 그림 (0.5초 노출).
    2.  SDK 내부 스크립트가 비동기(Async)로 **인증 서버에 ==현재 Origin(`http://192.168...`) 검증 요청**==을 보냄.
    3.  Whitelist 미등록으로 **401 Unauthorized** 응답 수신.
    4.  SDK가 이 응답을 감지하고 **프로그래매틱하게 화면을 덮는(Dimmed) 차단 로직**을 실행한 것으로 추측됨.

### 2.2 iOS 성공 원인: Custom Scheme & Header Spoofing
-  iOS `WKWebView`는 로컬 리소스 서빙을 위해 ==**`custom://localhost`*== 스킴을 사용.
- (보안 정책상 `http/https` 프로토콜에 대한 **Native Intercept(요청 가로채기)**를 막고 있음)

- **성공 메커니즘:**
    1.  **Request:** 웹뷰 내부에서 네이버 지도 리소스 및 인증 요청 발생.
    2.  **Header Spoofing:** Native Interceptor가 Outbound 요청의 **`Origin`** 헤더를 화이트리스트에 등록된 **`custom://localhost`**로 **강제 변조(Overwrite)**하여 전송.
    3.  **Result:** ==인증 서버는 요청이 `localhost`에서 온 것으로 인식==하여 승인(200 OK) -> ==SDK 차단 로직이 실행되지 않음==.

### 2.3 검증 (Verification)
- **Android 수정:** Android 네이티브 코드에서도 iOS와 동일하게 네트워크 인터셉터를 구현하여, `Origin` 헤더를 `http://localhost`로 강제 변조하도록 설정.

- **결과:** ==IP(`192.168.x.x`)로 접속했음에도 불구하고 네이버 지도가 **정상 작동함**==을 확인. (SDK가 변조된 헤더 덕분에 인증을 통과함)

## 3. 💡 인사이트 (Key Takeaways)

- **Symptom Analysis:** "화면이 안 나온다"는 현상을 분석할 때, 아예 안 나오는지(Network/CORS) vs 잠깐 나왔다 사라지는지(App Logic/Auth)를 구분하는 **관찰(Observation)**이 문제 해결의 핵심임.

- **Environment Differences:** 하이브리드 앱 디버깅 시, Android와 iOS의 ==**웹뷰 보안 정책 차이(HTTP Intercept 가능 여부)**로 인해 전혀 다른 동작 결과가 나올 수 있음==을 인지함.

- **Header Spoofing Utility:** 로컬 개발 환경(유동 IP)에서 화이트리스트 기반의 외부 API를 테스트할 때, 매번 IP를 등록하는 대신 ==**Native Level의 Header Spoofing**이 유용한 우회 전략이 될 수 있음.==

- [의문] iOS에서는 http://locahost가 아닌 custom://localhost로, 커스텀 프로토콜 스키마를 사용하였는데도 네이버지도 SDK 인증에 성공함. 화이트 리스트 origin에서 프로토콜은 체크를 안하고 있는건지 의문