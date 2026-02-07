## 1) “SWR 적용 후보”에서 SWR이 뭐야?

여기서 **SWR은 두 가지 의미로 같이 쓰여** (현업에서도 헷갈림):

### A. 원래 의미: **Stale-While-Revalidate(=SWR) 캐싱 전략**

- **stale(조금 오래된) 데이터를 일단 즉시 보여주고**
- **백그라운드에서 서버에 재검증(revalidate) 요청을 보내서**
- 새 응답이 오면 **캐시/화면을 갱신**하는 패턴이야. ([developer.mozilla.org](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Cache-Control?utm_source=chatgpt.com "Cache-Control header - HTTP - MDN Web Docs"))  
   HTTP 레벨에선 `Cache-Control: stale-while-revalidate=...` 같은 디렉티브로도 설명돼. ([developer.mozilla.org](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Cache-Control?utm_source=chatgpt.com "Cache-Control header - HTTP - MDN Web Docs"))

### B. 구현 수단: Vercel의 React 데이터 패칭 라이브러리 **SWR**

- 위 전략을 React에서 쓰기 쉽게 만든 “데이터 패칭 훅 라이브러리”도 이름이 SWR이야. ([코딩하지아](https://aboveimagine.tistory.com/133?utm_source=chatgpt.com "[React.js] 리액트의 네트워크 상태 관리 훅, SWR - 코딩하지아"))

### 커머스에서 “변동 낮은 응답”에 SWR을 붙인다는 뜻

예: **브랜드 메타/카테고리 트리/기획전 구성/리뷰 요약/추천 슬롯 구성(짧은 TTL)**  
→ 유저는 즉시 화면을 보고(캐시)  
→ 뒤에서 최신 데이터로 갱신(재요청)  
→ “로딩 스피너/빈 화면” 시간을 줄이는 목적. ([developer.mozilla.org](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Cache-Control?utm_source=chatgpt.com "Cache-Control header - HTTP - MDN Web Docs"))

---

## 2) “코드 설계” 자체가 P1로 자리잡을 수 있는 주제들 (아이디어 + 액션 플랜 씨앗)

너가 말한 “P1 설계(트레이드오프)”를 **코드 레벨 설계**로 더 쪼개면, 아래가 바로 P1 백로그가 돼. (P0의 ‘안 깨짐’이 아니라 **속도/비용/확장/개발생산성**을 두고 선택하는 것들)

### A. 상태/서버상태 설계 (웹+RN 공통)

**주제(트레이드오프)**: “클라 상태 vs 서버 상태”, “동기화 비용 vs 개발 속도”

- 액션 아이디어
  - 화면별 상태를 **UI 상태 / 서버 상태 / 폼 상태**로 분리(소유권 정의)
  - 서버 상태는 “캐시 키”, “무효화 규칙”, “staleTime/TTL”을 표로 관리
  - 장바구니/쿠폰/결제는 **optimistic 허용 범위**를 명시(실패 시 롤백 UX 포함)

### B. 데이터 경계 설계 (BFF/도메인 API)

**주제**: “서버에서 조합(BFF) vs 클라에서 조합”, “요청 수 vs 서버 복잡도”

- 액션 아이디어
  - 홈/상세에서 **N개의 API 호출**을 “조합 API 1개”로 줄일지 판단표 만들기
  - API 응답을 “캐시 가능/불가” 라벨링 + 캐시 키 설계(쿼리/유저 세그먼트)

### C. 캐시 무효화 설계 (진짜 P1)

**주제**: “캐시로 빨라짐 vs 데이터 신선도/정합성 리스크”

- 액션 아이디어
  - 리소스별로 “변경 트리거”를 정의  
     예) 상품상세: 가격변경/품절/쿠폰적용/프로모션 시작
  - 무효화 전략 3종 중 선택:
    - TTL 기반(단순) / 이벤트 기반(정교) / 하이브리드(대부분 현실적)
  - “stale-if-error(서버 장애 시 stale 허용)” 같은 **장애 완충 전략**도 후보로 둠(HTTP 확장 개념) ([datatracker.ietf.org](https://datatracker.ietf.org/doc/html/rfc5861?utm_source=chatgpt.com "RFC 5861 - HTTP Cache-Control Extensions for Stale Content"))

### D. 렌더 아키텍처 설계 (성능 ↔ 유지보수)

**주제**: “컴포넌트 분해 vs 렌더 비용”, “재사용성 vs 번들/복잡도”

- 액션 아이디어
  - 리스트/상세의 **핵심 컴포넌트 3개**를 선정해서:
    - props 표준화(데이터 shape 안정화)
    - 불필요 렌더 제거(메모이제이션/selector/분리)
    - 이미지/텍스트 우선순위(초기 LCP 대상 명시)

### E. 실험/플래그 설계 (KPI 최적화형 설계)

**주제**: “빠른 실험 속도 vs 코드 복잡도”

- 액션 아이디어
  - 추천/검색/상세 UI는 **기능 플래그 기본 탑재**(on/off, rollout %)
  - KPI가 걸린 변경은 “측정 이벤트 스키마”를 PR에 고정(추천 CTR, 검색 전환율, 결제 이탈률)

---

## 너가 바로 액션 플랜으로 만들기 쉬운 형태(예시 백로그 6개)

1. “우리 API 20개”를 **캐시 가능/불가/짧게/길게** 4분류 표 만들기 (+근거: 정합성 영향)
2. 검색/상세에서 **SWR 후보 3개** 선정: TTL, 캐시 키, 무효화 트리거까지 적기
3. 홈 진입 API 호출 수 측정 → “조합 API(BFF) 필요 후보 1개” 뽑기
4. 장바구니 수량 변경: optimistic 허용/불가 케이스를 문장으로 명시 + 실패 UX 표준화
5. 추천 슬롯: 플래그 + KPI 이벤트(CTR) 스키마를 컴포넌트 API에 포함
6. 캐시 장애 전략: “서버 오류 시 stale로 버티는가?” 화면 1개만 시범 적용(리스크 낮은 곳부터)
