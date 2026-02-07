> [!NOTE] 이 문서의 목적
> 애슬러 앱 코드베이스 분석 + 2025~2026 프론트엔드 채용 트렌드를 기반으로 발굴한 **새 액션 아이디어** 모음.
> [ACTION.md](ACTION.md) 에 이미 있는 항목은 포함하지 않는다.
> 검토 후 ACTION.md로 옮길 항목을 골라 사용한다.
>
> 각 항목에는 출처 태그를 표시한다:
> - `[코드분석]` : 애슬러 앱 코드를 직접 분석하여 발견한 개선점
> - `[채용트렌드]` : 주요 IT 기업 채용공고에서 가져온 트렌디 역량
> - `[복합]` : 코드 분석 + 트렌드 모두 해당

# P0) 전문성과 안정성

## (2) 코드 관련

### TypeScript 엄격 모드 단계적 강화 `[코드분석]`

현재 `tsconfig.base.json`에 `strict: true`가 있지만, 추가 엄격 옵션이 빠져 있어 암묵적 any 등이 허용된다.

- [ ] `noImplicitAny: true` 활성화 — 암묵적 any 타입 차단. 런타임에서야 발견되는 타입 에러 예방
- [ ] `noUnusedLocals: true` 활성화 — 사용하지 않는 변수 자동 감지. 죽은 코드 제거
- [ ] `noUnusedParameters: true` 활성화 — 사용하지 않는 함수 파라미터 감지
- [ ] `noImplicitReturns: true` 활성화 — 모든 코드 경로에서 반환값 보장. 예상치 못한 undefined 방지
- [ ] 패키지별 점진적 적용: `ui-system` → `lib-business` → `package-athler-js` → 앱 레벨
- [코드_레벨_개선_액션플랜 §1](docs/etc/코드_레벨_개선_액션플랜.md)

---

### 테스트 커버리지 임계값 설정 & CI 강제 `[코드분석]`

현재 `package-athler-js` 기준 테스트 파일 38개 / 소스파일 ~1,595개 (약 2.4%). `--passWithNoTests` 때문에 테스트 없는 패키지도 CI 통과.

- [ ] jest.config에 `coverageThreshold` 설정 (시작: statements 30% → 목표: 50%)
- [ ] `--passWithNoTests` 옵션을 핵심 패키지(`package-athler-js`, `lib-business`)에서 제거
- [ ] 커버리지 미달 시 PR 차단하는 CI 작업 추가
- [ ] 우선 테스트 대상 선정:
  - API query 함수 (`query-fn/*`) — 서버 통신 정합성
  - zustand 스토어 — 상태 관리 로직
  - 비즈니스 유틸리티 함수 — 계산/변환 로직
  - 에러 핸들링 경로 — 장애 대응 코드
- [코드_레벨_개선_액션플랜 §2](docs/etc/코드_레벨_개선_액션플랜.md)

---

### 순환 의존성 CI에서 차단하기 `[코드분석]`

`CircularDependencyPlugin`이 webpack에 있지만 warning만 출력. `home-display.tsx`, `header-token` 등 알려진 순환 참조가 이미 존재하고 TODO 주석도 4개.

- [ ] 현재 순환 참조 전체 목록 추출 및 영향도 평가
- [ ] 기존 알려진 순환 참조 리팩터링 (브릿지 모듈 패턴)
  - `home-display.tsx` → `@athlerjs/store/react-query` 순환
  - `header-token` → API index export 순환
- [ ] `CircularDependencyPlugin`에 `failOnError: true` 설정 (개발 환경)
- [ ] CI에서 새 순환 의존성 추가를 차단하는 작업 추가
- [코드_레벨_개선_액션플랜 §3](docs/etc/코드_레벨_개선_액션플랜.md)

---

### Error Boundary를 라우트/영역별로 확장하기 `[코드분석]`

현재 전체 앱에 Sentry `ErrorBoundary` 1개만 존재(App.tsx). 하나의 SDUI 컴포넌트 에러가 전체 앱을 크래시시킬 수 있음.

- [ ] 주요 라우트별(상품, 프로모션, 주소목록 등) ErrorBoundary 추가
- [ ] SDUI 디스플레이 컴포넌트 렌더링 영역에 ErrorBoundary 추가 (해당 영역만 숨김/대체)
- [ ] Braze 인앱 메시징 영역에 별도 ErrorBoundary 추가 (핵심 아닌 영역이므로 조용히 숨김)
- [ ] 에러 타입별 계층 구조 설계:
  - `AppError` (base) → `NetworkError` / `ValidationError` / `ServerError` / `BusinessError`
- [ ] 에러 타입별 UX 정의: 에러 페이지 vs 모달 vs 토스트 vs 고객센터 이동
- [코드_레벨_개선_액션플랜 §4](docs/etc/코드_레벨_개선_액션플랜.md)

---

### React.memo/useCallback 체계적 적용 전략 수립 `[코드분석]`

129개 디스플레이 컴포넌트에 체계적 메모이제이션 전략이 없음. 425개 useEffect 대비 최적화가 부족.

- [ ] 디스플레이 컴포넌트에 `React.memo` 적용 기준 정의
  - props가 안정적이고 부모 리렌더가 빈번한 컴포넌트 우선
- [ ] SDUI 스키마 변환 시 `useMemo` 적용 (무거운 연산)
- [ ] 리스트 아이템에 `React.memo` 적용 (스크롤 성능)
- [ ] 이벤트 핸들러에 `useCallback` 적용 (메모이즈된 자식에 전달 시)
- [ ] React Compiler가 이를 자동화할 수 있으므로 장기적으로 React Compiler 도입도 검토
- [코드_레벨_개선_액션플랜 §5](docs/etc/코드_레벨_개선_액션플랜.md)

---

### Zustand 스토어 정리 & 패턴 통일 `[코드분석]`

14개 zustand 스토어가 각 라우트에 산재. 서버 상태(React Query)와 UI 상태(Zustand) 경계 불명확.

- [ ] 14개 스토어가 관리하는 상태를 서버 상태 vs UI 상태로 분류 (상태 감사)
- [ ] 서버 상태인데 Zustand에 있는 것 → React Query로 이전
- [ ] 스토어 컨벤션 통일:
  - 파일명: `*.store.ts`
  - 위치: 공통 → `src/store/`, 라우트 전용 → 해당 라우트 폴더 내
  - 네이밍: `use[도메인]Store` 패턴
- [ ] 스토어 팩토리 또는 미들웨어 표준화 (devtools, persist, 로깅)
- [코드_레벨_개선_액션플랜 §6](docs/etc/코드_레벨_개선_액션플랜.md)

---

### ESLint 규칙 추가 강화 `[코드분석]`

좋은 기본 규칙이 있지만 몇 가지 중요한 규칙이 빠져 있음.

- [ ] `@typescript-eslint/no-floating-promises` 추가 — 처리되지 않은 Promise rejection 방지
- [ ] `@typescript-eslint/explicit-return-types` 추가 — API/스토어 함수에 반환 타입 명시
- [ ] `react-hooks/exhaustive-deps` 강화 — useEffect 의존성 배열 실수 방지 (2회 사고 이력)
- [ ] 패키지별 import 제한 규칙 — lib-business에서 UI import 금지
- [ ] 패키지별 ESLint 프리셋 분리 (앱용, 라이브러리용, API 패키지용)
- [코드_레벨_개선_액션플랜 §7](docs/etc/코드_레벨_개선_액션플랜.md)

---

### React Query 쿼리 키/옵션 표준화 `[코드분석]`

360개 이상의 useQuery/useMutation 호출이 있으나 staleTime/gcTime 전략이 통일되지 않음.

- [ ] API 데이터 특성별 staleTime/gcTime 표준 정의:
  - 자주 바뀌는 데이터(재고, 가격): staleTime 0 / gcTime 5분
  - 가끔 바뀌는 데이터(상품 정보): staleTime 5분 / gcTime 30분
  - 거의 안 바뀌는 데이터(카테고리): staleTime 30분 / gcTime 1시간
  - 사용자별 데이터(장바구니): staleTime 1분 / gcTime 10분
- [ ] `keepPreviousData: true` 적용 (리스트 페이지네이션 깜빡임 방지)
- [ ] 쿼리 키 팩토리 패턴 도입 (키 관리 일관성)
- [ ] `prefetchQuery`로 다음 화면 데이터 선제 로딩 전략
- [ ] in-flight deduplication 패턴 점검
- [코드_레벨_개선_액션플랜 §10](docs/etc/코드_레벨_개선_액션플랜.md)

---

### useEffect 의존성 배열 린트 & 가이드 강화 `[코드분석]`

> ACTION.md에 이미 "useEffect에 잘못된 값이 들어가는 것에 대한 강력한 경고"가 있음.
> 여기서는 **구체적 실행 방법**을 추가한다.

- [ ] ESLint `react-hooks/exhaustive-deps`를 `error` 레벨로 상향 (현재 warn인 경우)
- [ ] useEffect 내부에서 API 호출하는 패턴을 React Query로 대체하는 가이드 작성
- [ ] useEffect 사용이 적절한 경우 vs 부적절한 경우 가이드 문서화
  - 적절: 이벤트 리스너, 타이머, 외부 라이브러리 연동
  - 부적절: 데이터 fetch, 파생 상태 계산, 이벤트 핸들러

---

### 에러 타입 계층 구조 설계 `[코드분석]`

현재 표준화된 에러 타입이 없어 에러 핸들링이 일관되지 않음. 17개의 TODO/FIXME 주석 중 일부가 에러 핸들링 관련.

- [ ] 커스텀 에러 클래스 계층 설계:
  ```
  AppError (base)
  ├── NetworkError (네트워크 연결 실패)
  ├── ApiError (서버 응답 에러)
  │   ├── ValidationError (400)
  │   ├── AuthError (401/403)
  │   └── ServerError (5xx)
  ├── BusinessError (비즈니스 규칙 위반)
  │   ├── PaymentError
  │   ├── CouponError
  │   └── StockError
  └── UIError (렌더링 에러)
  ```
- [ ] 에러 타입별 복구 전략 매핑 (재시도/모달/토스트/에러페이지/고객센터)
- [ ] Sentry에 에러 타입별 태깅으로 분류/집계 가능하게 설정

---

## (3) 트렌디한 개발자 되기

### React Compiler 이해 및 적용 검토 `[채용트렌드]`

2025년 10월 v1.0 출시. 수동 `useMemo`/`useCallback`을 자동으로 대체하는 컴파일러.

- [ ] React Compiler가 자동 메모이제이션을 어떻게 하는지 학습
  - 공식 문서: react.dev/learn/react-compiler
- [ ] 기존 수동 `useMemo`/`useCallback` 코드 중 React Compiler가 대체할 수 있는 부분 파악
- [ ] 애슬러 앱에 React Compiler 적용 가능성 검토 (React 19 + RN 0.81 호환성)
- [ ] 적용 시 기존 수동 메모이제이션 코드 정리 계획
- [2026_프론트엔드_트렌드_채용공고_분석](docs/etc/2026_프론트엔드_트렌드_채용공고_분석.md)

---

### Playwright E2E 테스트 체계화 `[복합]`

2026년 기업 E2E 테스트 표준이 Cypress → Playwright로 전환됨. 멀티 브라우저 네이티브 지원, 병렬 실행, 더 안정적.

> ACTION.md에 페이지 어드민 Playwright 테스트가 이미 있음. 여기서는 **앱 전체 확장**을 다룬다.

- [ ] 핵심 커머스 플로우에 Playwright E2E 테스트 작성:
  - 결제 플로우 (상품 선택 → 장바구니 → 결제)
  - 로그인 플로우 (소셜 로그인 → 메인 화면)
  - 쿠폰 적용 플로우 (쿠폰 선택 → 금액 반영)
- [ ] 멀티 브라우저(Chrome, Firefox, Safari/WebKit) 테스트 환경 구축
- [ ] CI에서 Playwright 테스트 자동 실행 (PR 체크)
- [ ] Playwright 테스트 모범 사례 적용:
  - `data-test` 속성 사용 (class/id 대신)
  - API 모킹으로 결정론적 테스트
  - 적절한 대기 전략으로 flaky 테스트 방지
- [2026_프론트엔드_트렌드_채용공고_분석 §2](docs/etc/2026_프론트엔드_트렌드_채용공고_분석.md)

---

### Visual Regression Testing 도입 `[채용트렌드]`

스크린샷 기반으로 UI 변경을 자동 감지. 디자인 시스템 안정성 확보에 핵심.

- [ ] Playwright 스크린샷 비교 기능을 활용한 Visual Regression 설정
- [ ] 핵심 페이지 비주얼 베이스라인 등록 (홈, 상품 상세, 장바구니, 결제)
- [ ] ui-system 컴포넌트별 비주얼 스냅샷 등록
- [ ] CI에서 비주얼 변경 감지 시 리뷰어에게 알림

---

### Core Web Vitals CI 통합 `[복합]`

CWV가 매출/SEO에 직결. 성능 예산을 정의하고 CI에서 자동 검증.

- [ ] Lighthouse CI 또는 유사 도구 도입
- [ ] 성능 예산(Performance Budget) 정의:
  - LCP < 2.5s (3G 환경 기준)
  - CLS < 0.1
  - INP < 200ms
  - 번들 초기 로딩 < 500KB
- [ ] PR마다 성능 점수 자동 체크 및 위반 시 경고
- [ ] 성능 회귀 방지: 이전 빌드 대비 악화 시 차단
- [2026_프론트엔드_트렌드_채용공고_분석 §2](docs/etc/2026_프론트엔드_트렌드_채용공고_분석.md)

---

### React Server Components(RSC) 개념 이해 `[채용트렌드]`

Next.js 채용공고의 71%가 요구. 서버/클라이언트 컴포넌트 분리 원칙.

- [ ] RSC의 핵심 개념 학습:
  - 서버 컴포넌트 vs 클라이언트 컴포넌트
  - "use client" / "use server" 지시문
  - 데이터 fetching 패턴 변화
- [ ] 애슬러 SDUI와 RSC의 공통점/차이점 분석:
  - 공통: 서버가 UI를 결정
  - 차이: RSC는 React 컴포넌트 트리 자체를 서버에서 렌더, SDUI는 JSON 스키마
- [ ] Streaming, Progressive Hydration 개념 이해
- [ ] 면접에서 "SDUI 경험을 RSC 맥락으로 설명"할 수 있도록 준비
- [2026_프론트엔드_트렌드_채용공고_분석 §2](docs/etc/2026_프론트엔드_트렌드_채용공고_분석.md)

---

### Concurrent React 패턴 익히기 `[채용트렌드]`

React 19의 핵심 기능. 사용자 경험을 크게 향상시키는 패턴들.

- [ ] `useTransition` 학습 — 긴 작업 중에도 UI 반응성 유지
  - 적용 시나리오: 검색 필터링, 탭 전환, 대량 리스트 렌더링
- [ ] `useOptimistic` 학습 — 낙관적 업데이트 패턴
  - 적용 시나리오: 장바구니 수량 변경, 좋아요, 쿠폰 적용
- [ ] `Suspense` 활용 — 비동기 데이터 로딩의 선언적 처리
  - 적용 시나리오: SDUI 컴포넌트 로딩, 상품 상세 데이터
- [ ] 애슬러 앱에서 가장 효과적인 적용 시나리오 3개 선정 및 적용

---

### Edge Computing 인식 `[채용트렌드]`

CDN 엣지에서 로직을 실행. Cloudflare Workers, Vercel Edge Functions 등.

- [ ] Edge Functions의 개념과 장단점 이해
  - 장점: 사용자에게 가까운 곳에서 실행 → 응답 시간 단축
  - 단점: 제한된 런타임, cold start
- [ ] 프론트엔드가 Edge에서 실행되는 것의 의미 파악
  - A/B 테스트를 Edge에서 분기
  - 이미지 최적화를 Edge에서 처리
  - API 응답 캐싱을 Edge에서 처리
- [ ] 애슬러에서 Edge가 유용할 수 있는 시나리오 고민 (웹 SSR 시)

---

### 코드 스플리팅 & 번들 최적화 실전 `[복합]`

webpack에 BundleAnalyzerPlugin이 있으나 적극 활용하지 않음. 대형 서드파티(Braze, Firebase 등) 별도 청크 분리 가능.

- [ ] `ANALYZE=true` 빌드로 현재 번들 구성 분석
- [ ] 라우트 기반 코드 스플리팅 적용:
  - 주소목록(`address-list`)
  - 상품(`product`)
  - 프로모션(`promotion-list`)
  - 등 주요 라우트별 청크 분리
- [ ] 대형 의존성 별도 청크 분리:
  - Braze 메시징 (대형 SDK)
  - Firebase Analytics
- [ ] tree-shaking이 제대로 되지 않는 부분 점검
- [ ] 번들 사이즈를 PR마다 추적하는 CI 작업 추가
- [ ] 청크별 사이즈 예산 설정
- [코드_레벨_개선_액션플랜 §9](docs/etc/코드_레벨_개선_액션플랜.md)

---

### 접근성(A11y) 기본기 `[채용트렌드]`

WCAG 2.1 AA가 이제 선택이 아닌 필수 역량. 글로벌 기업은 법적 의무, 한국도 장애인차별금지법 적용.

- [ ] 시맨틱 마크업 기본 학습 (적절한 HTML 요소 사용)
- [ ] 포커스 관리 학습 (모달, 드롭다운 등에서 포커스 트래핑)
- [ ] 키보드 네비게이션 기본 (Tab, Enter, Escape 핸들링)
- [ ] ARIA 속성 기본 (aria-label, aria-describedby, role 등)
- [ ] 디자인 시스템(`ui-system`)에 접근성 기본 규칙 적용
- [ ] 자동 접근성 검사 도구 도입 (eslint-plugin-jsx-a11y 등)
- [ ] React Native에서의 접근성: `accessible`, `accessibilityLabel` 등

---

### Monorepo 최적화 `[복합]`

Yarn Berry 4.1.0 + 7개 패키지 운영 중. 더 효율적으로 활용할 수 있는 방법.

- [ ] 빌드 캐싱 전략 점검 (변경된 패키지만 빌드)
- [ ] 의존성 호이스팅 최적화
- [ ] workspace 간 의존성 그래프 시각화 및 정리
- [ ] 패키지 간 API 경계 명확화 (불필요한 re-export 제거)
- [ ] 새 패키지 추가 시 체크리스트 문서화

---

### AI 도구 활용 워크플로우 고도화 `[복합]`

> ACTION.md에 AI 사용하기가 이미 있음. 여기서는 **실전 워크플로우 구체화**를 다룬다.

- [ ] PR 리뷰에 AI 코드 리뷰 자동화 도입 (SuperClaude code-reviewer 활용)
- [ ] 코드 생성 → 자동 테스트 → 자동 린트 파이프라인 구축
- [ ] AI가 생성한 코드의 품질 게이트 정의 (테스트 커버리지, 타입 안전성, 번들 크기)
- [ ] 반복적인 SDUI 엘리먼트 추가 작업 AI 자동화

---

# P1) 비즈니스 임팩트형 성능/UX

### React Query 최적화 전략 수립 `[코드분석]`

360개 이상의 useQuery/useMutation 호출. staleTime/gcTime 전략 불통일. keepPreviousData 미활용.

- [ ] API 데이터 특성별 쿼리 옵션 표준 수립 (위의 P0 코드 관련 React Query 항목과 동일)
- [ ] 중복 요청 제거 패턴 정리 (같은 데이터를 여러 컴포넌트가 동시 요청하는 경우)
- [ ] 쿼리 무효화(invalidation) 전략 정리 (mutation 후 어떤 쿼리를 무효화할지)
- [ ] React Query Devtools 활용법 팀 공유

---

### 번들 사이즈 모니터링 CI 통합 `[코드분석]`

현재 BundleAnalyzerPlugin이 있지만 수동 분석만 가능. 번들 크기 추적 자동화 필요.

- [ ] CI에서 빌드 후 번들 크기를 자동으로 측정하는 작업 추가
- [ ] PR별 번들 크기 변화를 코멘트로 보여주는 봇 설정
- [ ] 번들 크기 예산 초과 시 경고 또는 차단
- [ ] 주요 청크별 사이즈 트렌드 추적

---

### 이미지 최적화 전략 `[채용트렌드]`

패션 커머스는 이미지가 핵심. 이미지 최적화가 LCP에 가장 큰 영향.

- [ ] WebP/AVIF 포맷 적용 가능성 검토
- [ ] responsive 이미지 (srcset, sizes) 적용
- [ ] lazy loading 전략 강화 (뷰포트 진입 시점 로딩)
- [ ] 이미지 CDN 최적화 (리사이징, 포맷 변환, 캐싱)
- [ ] placeholder/skeleton 이미지로 CLS 방지

---

### 성능 예산(Performance Budget) 정의 `[채용트렌드]`

성능을 "감"이 아닌 "숫자"로 관리. 예산을 넘으면 경고.

- [ ] 웹 성능 예산 정의:
  - 초기 번들: < 500KB
  - 총 번들: < 2MB
  - LCP: < 2.5s (3G)
  - 컴포넌트당 번들: < 50KB
- [ ] RN 성능 예산 정의:
  - Cold start: < 3s
  - 화면 전환: < 300ms
  - 스크롤 프레임 드롭: < 5%
  - JS 스레드 점유율: < 30%
- [ ] 예산 위반 시 CI에서 경고하는 자동화 설정

---

# P1) 설계 트레이드오프 판단

### 서버 상태 vs UI 상태 경계 명확히 분리하기 `[코드분석]`

14개 zustand 스토어 + 360개 React Query 호출. 어디에 무엇을 두어야 하는지 기준이 필요.

- [ ] 상태 분류 원칙 정의:
  - **서버 상태 (React Query)**: API에서 오는 데이터, 캐싱/동기화가 필요한 것
  - **UI 상태 (Zustand)**: 모달 열림/닫힘, 폼 입력, 스크롤 위치 등 순수 클라이언트
  - **URL 상태**: 페이지네이션, 필터, 정렬 (URL params)
  - **폼 상태**: 폼 라이브러리 (react-hook-form 등)
- [ ] 현재 양쪽에 중복 존재하는 상태 감사(audit) 및 정리
- [ ] 상태 분류 가이드 문서 작성 (새 기능 개발 시 참조용)
- [swr_설계_트레이드오프_판단_액션_플랜](docs/etc/swr_설계_트레이드오프_판단_액션_플랜.md)

---

### 캐싱 5단 레이어 설계 `[채용트렌드]`

> GUIDELINE.md에서 이미 언급된 "5단 캐시 레이어"를 구체화한다.

- [ ] 5단 캐시 레이어 정리:
  1. **HTTP 캐시**: Cache-Control, ETag — 브라우저 기본
  2. **CDN 캐시**: 이미지, 정적 에셋 — CloudFront 등
  3. **클라이언트 데이터 캐시**: React Query — API 응답 캐싱
  4. **이미지 캐시**: RN FastImage, 웹 lazy loading — 이미지 전용
  5. **사전 계산**: 서버에서 미리 계산한 데이터 — SDUI 스키마 등
- [ ] API별 어떤 캐시 레이어를 사용하는지 매핑
- [ ] 캐시 무효화(invalidation) 전략 정리

---

### 요청 최적화 전략 `[코드분석]`

> GUIDELINE.md의 "요청 수 줄이기: 배칭, 중복 요청 제거, prefetch" 를 구체화.

- [ ] Request Batching: 여러 API를 하나로 묶을 수 있는 곳 탐색
- [ ] in-flight deduplication: 같은 요청이 동시에 나가는 것 방지 (React Query 기본 지원 확인)
- [ ] Prefetch: 다음 화면 데이터 선제 로딩
  - 상품 리스트 → 상품 상세 prefetch
  - 장바구니 → 결제 정보 prefetch
- [ ] 불필요한 API 호출 제거 (같은 데이터를 여러 번 요청하는 경우)

---

# P2) 운영/관측(Observability) & 릴리즈

### CI/CD 파이프라인 강화 `[코드분석]`

현재 `pr-checks.yml`에 기본 lint/test/build만 존재. Node 18 사용(프로젝트는 >=22 요구).

- [ ] Node 22+ 업그레이드 (CI 환경)
- [ ] lint/typecheck/test를 병렬 작업으로 분리 (CI 속도 향상)
- [ ] typecheck 전용 작업 추가 (`tsc --noEmit`)
- [ ] 코드 커버리지 리포팅 추가 (Codecov 또는 유사 서비스)
- [ ] PR 코멘트에 커버리지 변화/번들 크기 변화 자동 표시
- [코드_레벨_개선_액션플랜 §8](docs/etc/코드_레벨_개선_액션플랜.md)

---

### 의존성 보안 스캐닝 자동화 `[코드분석]`

많은 서드파티 SDK 사용(Firebase, Sentry, Amplitude, Braze 등). 보안 취약점 모니터링 필요.

- [ ] `yarn audit`를 CI에 통합 (빌드 시 자동 실행)
- [ ] GitHub Dependabot 설정 (자동 보안 PR 생성)
- [ ] 주기적인 의존성 업데이트 루틴 수립 (월 1회 등)
- [ ] 의존성 목록 문서화 (왜 각 라이브러리가 필요한지)

---

### 번들 사이즈 PR별 추적 `[코드분석]`

번들 크기 변화를 모니터링하지 않으면 점진적으로 비대해짐.

- [ ] CI에서 빌드 후 번들 사이즈 자동 측정
- [ ] 이전 빌드 대비 변화량을 PR 코멘트로 표시
- [ ] 임계값 초과 시 경고 (예: 10% 이상 증가)
- [ ] 월간 번들 크기 트렌드 리포트

---

# P2) 보안/리스크 방어

### ESLint 보안 규칙 강화 `[코드분석]`

> ACTION.md의 보안 섹션이 Coming Soon이었으므로, 코드 레벨에서 즉시 할 수 있는 것들.

- [ ] `dangerouslySetInnerHTML` 사용 차단/경고 규칙 추가
- [ ] `eval()` / `Function()` 생성자 사용 금지 규칙
- [ ] 민감 정보 하드코딩 탐지 규칙 강화 (API 키 패턴 매칭)
- [ ] `innerHTML`, `outerHTML` 직접 사용 금지
- [ ] `postMessage` origin 검증 필수 규칙 (WebView 보안)

---

### 의존성 감사 자동화 `[코드분석]`

22개 이상의 analytics SDK 관련 export가 `lib-business`에 존재. 노출 영역이 넓음.

- [ ] `lib-business` export 감사 — 불필요한 노출 줄이기 (facade 패턴)
- [ ] 서드파티 SDK 도입 시 보안 심사 체크리스트 작성:
  - 라이선스 확인
  - 알려진 취약점 확인
  - 번들 크기 영향
  - 데이터 수집 범위
  - 대안 존재 여부
- [ ] `yarn audit` 주기적 실행 및 결과 리뷰

---

# P3) 차별화

> ACTION.md에 P3 섹션이 아직 없으므로, GUIDELINE.md의 P3 정의에 맞는 아이디어를 미리 준비.

### 접근성(A11y) 전문성 `[채용트렌드]`

GUIDELINE.md P3: "접근성 기반 해자. 아는 사람이 적어 희소."

- [ ] 컴포넌트 레벨 접근성 규칙 정립 (포커스/라벨/키보드)
- [ ] 자동 접근성 검사 + 수동 점검 루틴 구축
- [ ] React Native에서의 접근성 구현 경험 (VoiceOver, TalkBack)
- [ ] 접근성 개선 전/후 사용성 테스트

---

### AI UX 실전 `[채용트렌드]`

GUIDELINE.md P3: "AI UX / 특수기술"

- [ ] 검색/추천에 AI 활용 가능성 탐구 (LLM 기반 상품 추천)
- [ ] CS 비용 절감을 위한 AI 챗봇 UX 설계
- [ ] AI DA 프로젝트를 "AI UX 사례"로 포트폴리오화
