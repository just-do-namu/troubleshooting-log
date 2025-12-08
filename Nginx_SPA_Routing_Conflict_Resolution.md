---
tags:
  - Nginx
  - CORS
  - ReverseProxy
  - Routing
  - SPA-Fallback
  - Troubleshooting
  - Infrastructure
date: 2025-12-08
---

🟢 [해결/Nginx/CORS] 갑작스런 CORS와 Nginx Reverse Proxy 설정 중 발생한 SPA 라우팅 충돌 및 CORS 해결 과정

> **Date:** 2025.12.08
> **Tags:** #Nginx #CORS #ReverseProxy #Routing #SPA-Fallback #Troubleshooting #Infrastructure

---

## 1. 🚨 증상 (Symptoms)

- **Issue 1 :** 서비스에서 정상적으로 작동중이던 외부 공공 데이터 API가 갑자기 **CORS(Cross-Origin Resource Sharing) Error** 발생하며 일부 서비스에 장애 발생==(해당 원인은 아직 불명).==
- **Action:** 클라이언트 사이드 호출을 중단하고, **Nginx Reverse Proxy**를 통해 우회하도록 설정 변경.
- **Issue 2 (Critical):** 프록시 설정 후 API를 호출했으나, JSON 데이터가 아닌 **==프로젝트의 `index.html` 소스 코드==가 200 OK로 반환됨.**

## 2. 🕵️‍♂️ 원인 분석 (Root Cause Analysis)

### 2.1 현상 분석: 왜 HTML이 반환되었는가?

- API 응답으로 `index.html`이 왔다는 것은, 요청이 백엔드(외부 API)로 전달되지 않고 **프론트엔드 서버(Nginx) 내부에서 처리**되었다는 의미임.
- 이는 SPA(Single Page Application)의 일반적인 라우팅 설정인 **Fallback(존재하지 않는 경로는 index.html로 연결)** 정책이 API 요청에도 적용되었기 때문으로 추론함.

### 2.2 심층 분석: Nginx 라우팅 우선순위

- **설정 검토:** `nginx.conf` 파일 확인 결과, API 프록시 설정(`location /openapi/`)이 SPA 리다이렉트 설정(`location /`)보다 ==**하단**에 작성==되어 있었음.
- **메커니즘:** Nginx의 `location` 블록 매칭은 ==작성 순서==뿐만 아니라 ==**Specific Match(구체적 일치)**==와 ==**Regex Match(정규식 일치)**== 간의 우선순위 규칙을 따름.
- **결론:** API 요청 패턴이 상단에 정의된 광범위한 SPA 라우팅 규칙(`try_files $uri $uri/ /index.html`)에 먼저 포착(Capture)되어 의도치 않은 리다이렉트가 발생함.

## 3. 🛠️ 해결 (Solution)

### Nginx Configuration Optimization

- API 프록시 설정을 **최상단**으로 이동시키고, 명확한 Prefix(`^~`)를 사용하여 우선순위를 확보함.

```nginx
# [AS-IS] (Bad: SPA 라우팅이 먼저 적용됨)
location / {
    try_files $uri $uri/ @rewrite;
}

location @rewrite {
    rewrite ^ /index.html;
}

location /openapi/ {
    proxy_pass [https://external-api.com/](https://external-api.com/);
}
```

```nginx
# [TO-BE] (Good: API 요청을 먼저 가로채서 프록시 처리)
location / {
    try_files $uri $uri/ @rewrite;
}

# 'location /openapi/'는 가장 긴 Prefix 매치로 이미 우선순위가 높지만,
# ^~ 를 사용해 정규식 검사(Regex)보다도 먼저 확정시킵니다.
location ^~ /openapi/ {
	proxy_pass https://external-api.com/;
}

location @rewrite {
    rewrite ^ /index.html last; # 'last' 지시어 추가
}
```

## 4. 💡 인사이트 (Key Takeaways)

- **Infrastructure Awareness:** 프론트엔드 개발자라도 **Web Server(Nginx)**의 라우팅 메커니즘을 이해해야 CORS와 같은 네트워크 이슈를 주도적으로 해결할 수 있음.
- **Debugging Pattern:** "API 응답으로 HTML이 온다"는 현상은 십중팔구 **라우팅 설정의 충돌(Routing Conflict)**임을 경험적으로 체득함.
