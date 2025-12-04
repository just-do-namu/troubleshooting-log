---
tags:
  - VueJS
  - OpenSource
  - Troubleshooting
  - Library-Analysis
  - Decision-Making
date: 2025-12-04
---
# 🟡 [해결/Library] 오픈소스 지도 라이브러리의 다중 인스턴스 충돌 분석 및 방어적 해결 전략

> **Date:** 2025.12.04  
> **Tags:** `#Vue.js` `#NaverMap` `#OpenSource` `#Troubleshooting` `#DecisionMaking`  
> **Related Issue:** [GitHub Issue #34](https://github.com/Dongkyuuuu/vue3-naver-maps/issues/34)

---

## 1. 🚨 증상 (Symptoms)

- **현상:** 한 화면에 두 개의 지도(Background Map + Modal Map)가 동시에 존재할 때, 모달 지도에 생성한 마커(Marker)가 엉뚱하게 백그라운드 지도에 렌더링되는 **컨텍스트 오염(Context Pollution)** 현상 발생.
- **상황:**
    - 메인 화면에는 지도가 깔려 있음 (Map A).
    - 팝업(Modal)을 띄우고 그 안에 지도를 로드함 (Map B).
    - Map B에 핀을 찍었는데, Map A에 핀이 나타남.

---

## 2. 🕵️‍♂️ 원인 분석 (Root Cause Analysis)

### 2.1. 라이브러리 내부 로직 분석 (Code Review)
문제가 발생한 오픈소스 라이브러리(`vue3-naver-maps`)의 소스 코드를 분석하여 구조적 원인을 파악함.

1.  **싱글턴 패턴의 오용:** 라이브러리가 내부적으로 지도 인스턴스를 관리할 때, 다중 인스턴스를 고려하지 않고 **전역 객체나 단일 레퍼런스**에 의존하는 경향이 있음.
2.  **초기화 시점 충돌:** 두 번째 지도(Map B)가 마운트될 때, 라이브러리가 현재 활성화된 지도 컨텍스트를 Map B로 덮어쓰지 못하거나, 마커 컴포넌트가 **'가장 먼저 생성된 지도(Map A)'**를 부모로 잘못 인식함.

### 2.2. 결론
- 해당 라이브러리는 **'한 화면에 하나의 지도'**라는 암묵적 전제하에 설계되었음.
- 따라서 다중 지도를 지원하려면 라이브러리 코드를 수정하거나, 애플리케이션 레벨에서 컨텍스트를 강제로 주입해야 함.

---

## 3. 🛠️ 해결 및 조치 (Solution)

단순한 버그 수정이 아니라, **운영 안정성(Stability)**을 최우선으로 한 단계별 접근을 시도함.

### Step 1. 이슈 제보 (Issue Reporting)
- **Action:** 라이브러리 저장소에 이슈([Issue #34](https://github.com/Dongkyuuuu/vue3-naver-maps/issues/34))를 생성하여 현상을 제보하고 원인을 공유함.
- **Result:** 메인테이너로부터 수정된 버전(v4.4 추정) 코드를 전달받음.

### Step 2. 샌드박스 테스트 (Verification)
- **Action:** 별도의 Playground 프로젝트에서 수정 버전을 테스트함.
- **Finding:** 버그는 해결되었으나, 내부 로직의 변경 범위가 넓어 기존 코드에 대한 **사이드 이펙트** 가능성이 높다고 판단됨.

### Step 3. 최종 의사결정: "방어적 코딩 (Defensive Coding)"
- **Decision:** 라이브러리 자체를 업그레이드하거나 `patch-package`로 코드를 뜯어고치는 것은 **운영 리스크(High Risk)**가 크다고 판단하여 보류.
- **Solution:** 라이브러리 의존성을 낮추고, 애플리케이션 레벨에서 **명시적으로 제어**하는 방식을 택함.
    ```javascript
    // AS-IS: 라이브러리가 알아서 마커를 지도에 붙여주길 기대함 (실패)
    <naver-marker :lat="lat" :lng="lng" />

    // TO-BE: 마커 생성 직후, 강제로 타겟 지도 인스턴스를 주입 (성공)
    onMounted(() => {
      const marker = new naver.maps.Marker({ ... });
      marker.setMap(targetMapInstance); // Explicit Binding
    });
    ```

---

## 4. 💡 인사이트 (Key Takeaways)

1.  **오픈소스 의존성 관리:** 라이브러리는 '블랙박스'가 아니며, 필요할 때는 내부 코드를 뜯어보고 동작 원리를 이해해야 함.
2.  **기술적 타협의 미학:** "최신 버전 업데이트"가 항상 정답은 아님. 비즈니스 안정성을 위해 **'완벽한 수정'보다 '안전한 우회'**를 선택하는 것이 더 나은 엔지니어링일 수 있음.
3.  **적극적인 커뮤니케이션:** 이슈 리포팅을 통해 생태계에 기여하고, 메인테이너와 소통하며 해결책을 찾는 과정의 중요성 체감.