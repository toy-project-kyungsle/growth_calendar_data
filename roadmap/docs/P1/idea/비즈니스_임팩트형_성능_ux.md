> **커머스 KPI(검색 전환율/결제 이탈률/추천 CTR)를 흔드는 병목을 찾아 제거하는 성능/UX 개선**

(웹은 **CWV(LCP/INP/CLS)**, RN은 **체감(시작/전환/스크롤/입력 응답)**을 같은 방식으로 다룬다.)

아래는 **바로 실행 가능한 액션 플랜(측정→원인→개선→회귀방지)**로 정리했어.
참고로 CWV는 현재 **LCP/INP/CLS**가 핵심이고, **INP가 FID를 2024-03-12에 대체**했어. ([web.dev](https://web.dev/blog/inp-cwv-launch?utm_source=chatgpt.com "Interaction to Next Paint is officially a Core Web Vital 🚀 | Blog"))

---

## 0) 타겟을 "페이지"가 아니라 "플로우"로 잡기 (KPI 직결)

성능을 이렇게 묶어야 비즈니스 임팩트가 커져.

### 액션

- **검색(Search)** → 검색 전환율
- **상품상세(PDP)** → PDP→장바구니/구매 전환
- **장바구니(Cart)** → 결제 진입률
- **체크아웃/결제(Checkout)** → 결제 이탈률
- **추천(Reco)** → 추천 CTR

✅ Done: 성능 개선 타겟이 플로우 단위로 정의됨.

---

## 1) 공통 1단계: "성능 지표를 KPI와 같이 본다"

### 웹

**필드 데이터(실사용자)**를 우선으로: Search Console CWV 리포트/CrUX로 "실제 유저 경험"을 본다. ([구글 도움말](https://support.google.com/webmasters/answer/9205520?hl=en&utm_source=chatgpt.com "Core Web Vitals report - Search Console Help"))

### 액션 (웹)

- (A) **Search Console에서 Poor/NI URL 그룹**을 플로우별로 분류(검색/PDP/체크아웃)
- (B) **CrUX(오리진/URL)**로 "모바일/데스크탑" 차이를 확인 ([Chrome for Developers](https://developer.chrome.com/docs/crux?utm_source=chatgpt.com "Overview of CrUX | Chrome UX Report"))

### RN

RN은 공식 가이드가 말하듯 **프로파일링(Dev mode off) + 도구(iOS Instruments/Android Profiler)**로 병목을 잡는 게 기본 루틴이야. ([React Native](https://reactnative.dev/docs/profiling?utm_source=chatgpt.com "Profiling"))

### 액션 (RN)

- (A) 체감 구간 4개만 고정 측정: **cold start, 화면 전환, 스크롤, 입력 반응**
- (B) 이 4구간에서 **JS thread/렌더 폭증/이미지/리스트** 중 뭐가 원인인지 분류

✅ Done: 성능 지표와 KPI를 같이 보는 루틴이 정착됨.

---

## 2) 웹(CWV) 액션 플랜: LCP/INP/CLS를 "플로우별 원인"으로 쪼개기

### A. LCP(로딩): "PDP/검색 첫 화면에서 가장 큰 요소(대개 이미지/히어로/상품리스트)가 늦는 문제"

### 액션

- **상품 이미지/썸네일**: 사이즈 최적화/지연 로드/우선순위 로드(첫 화면만)
- **서버 응답/렌더 경로**: 초기 HTML/데이터가 늦으면 LCP가 통째로 밀림(SSR/캐시/데이터 분리로 해결 후보)
- **3rd-party 스크립트**: 마케팅 태그가 메인 스레드를 막아 LCP 악화시키는 케이스를 항상 의심

(웹.dev는 CWV 개선을 "가장 현실적/임팩트 큰 최적화부터" 하라고 정리해 둠) ([web.dev](https://web.dev/articles/top-cwv?utm_source=chatgpt.com "The most effective ways to improve Core Web Vitals | Articles"))

✅ Done: LCP 병목이 플로우별로 식별됨.

### B. INP(반응성): "클릭/탭 했는데 다음 페인트까지 늦는 문제(체감 렉)"

INP가 핵심 CWV로 들어왔고(=상호작용 성능이 SEO/UX 모두 중요), 이제 "클릭 렉"을 제대로 다뤄야 해. ([web.dev](https://web.dev/blog/inp-cwv-launch?utm_source=chatgpt.com "Interaction to Next Paint is officially a Core Web Vital 🚀 | Blog"))

### 액션

- **이벤트 핸들러에서 무거운 작업 제거**: 클릭→동기 JS 연산/큰 상태 업데이트/렌더 폭증이 원인
- **렌더 범위 축소**: "장바구니 담기" 같은 액션에서 페이지 전체 리렌더가 일어나면 INP가 터짐
- **상호작용 분리**: 클릭 직후 꼭 필요한 UI만 즉시 업데이트하고, 나머지는 뒤로 미룸(비동기/지연 렌더)

✅ Done: INP 개선 액션이 정의됨.

### C. CLS(레이아웃 흔들림): "검색/PDP에서 이미지/폰트/리뷰/추천 섹션이 뒤늦게 들어오며 흔들림"

### 액션

- 이미지/배너 영역 **고정 높이(placeholder)** 확보
- 가격/쿠폰 영역처럼 변동 큰 UI는 **레이아웃 슬롯을 미리 잡아두기**
- 폰트 로딩으로 흔들리면 폰트 전략 조정(FOIT/FOUT 최소화)

✅ Done: CLS 개선 액션이 정의됨.

---

## 3) RN(체감 성능) 액션 플랜: '4구간'별로 원인→해결을 고정

RN 공식 성능/프로파일링 문서가 말하는 핵심은: **개발 모드 끄고, 병목을 측정 도구로 확인하라**야. ([React Native](https://reactnative.dev/docs/profiling?utm_source=chatgpt.com "Profiling"))

### A. Cold start(앱 시작)

### 액션

- 초기 진입에 필요 없는 모듈/화면/데이터는 **지연 로드**
- 앱 시작 시 네트워크를 "즉시 필수 vs 나중"으로 나누기(초기 경쟁 제거)

✅ Done: Cold start 개선 액션이 정의됨.

### B. 화면 전환(네비게이션)

### 액션

- 전환 시 실행되는 API 호출/상태 초기화/리스트 렌더를 분리
- "전환 애니메이션 동안 JS가 바쁘지 않게" 작업을 뒤로 미루는 설계(전환 직후 몰아치지 않기)

✅ Done: 화면 전환 개선 액션이 정의됨.

### C. 스크롤 잔렉(리스트)

커머스 RN 성능의 80%는 리스트/이미지야.

### 액션

- 리스트 셀 컴포넌트 **가볍게**: 불필요한 props/리렌더 제거
- 이미지: 해상도/캐시/리사이즈 정책 점검
- "필터/정렬 바뀔 때 전체 리스트 재마운트" 같은 패턴 제거

✅ Done: 스크롤 잔렉 개선 액션이 정의됨.

### D. 입력 반응(검색창/쿠폰 입력)

### 액션

- 입력마다 API 호출 금지: debounce/throttle + "확정 액션(검색 버튼)" 병행
- 입력에 의해 발생하는 렌더 범위 축소(검색창 입력이 페이지 전체 state를 흔들지 않게)

✅ Done: 입력 반응 개선 액션이 정의됨.

---

## 4) "성과로 연결되는" 실전 우선순위 (커머스 기준)

아래 순서로 때리면 KPI에 가장 잘 연결돼.

### 액션 (우선순위)

1. **체크아웃(결제 이탈률)**: 클릭 렉(INP) + 결제 단계 렌더 폭증 + 불필요 호출 제거
2. **검색(검색 전환율)**: 첫 결과 노출(LCP) + 입력 반응(INP/체감) + 요청 폭탄 방지
3. **PDP(전환)**: 이미지 LCP + 레이아웃 안정(CLS) + '담기/구매' 클릭 렉(INP)
4. **추천(CTR)**: 노출까지의 속도 + 스크롤 중 잔렉 제거(보이는 타이밍이 CTR에 영향)

✅ Done: KPI 직결 우선순위가 정해짐.

---

## 5) 회귀 방지 (=P1을 "지속 성과"로 만드는 장치)

이게 없으면 성능은 늘 되돌아가.

### 액션

- **성능 예산(Performance budget)**: "PDP LCP/INP/CLS, RN 리스트 스크롤 기준"을 숫자로 박아 PR에서 체크
- **고위험 구간 PR 규칙**: 결제/쿠폰/검색 입력/리스트 관련 변경은 "리렌더 증가/호출 증가" 근거 제출
- **릴리즈 게이트(간단)**: 새 릴리즈에서 "체크아웃 성공률/결제 이탈률"이 흔들리면 즉시 되돌리는 룰(정책)

✅ Done: 성능 회귀 방지 장치가 정착됨.
