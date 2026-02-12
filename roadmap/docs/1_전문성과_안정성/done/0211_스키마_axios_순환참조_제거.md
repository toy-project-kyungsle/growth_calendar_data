
**브랜치**: `kill-circle-import-0210`
**커밋**: `2ed10b5e8` — "Update : schema 및 user auth 순환참조 제거"
**작성일**: 2026-02-11

---

## 1. 개요: 무엇을 해결하기 위해 개발하였는가

두 영역에서 발생한 **순환 참조(Circular Dependency)** 를 제거하기 위한 작업입니다.

| 영역 | 위치 | 문제 |
|------|------|------|
| **Schema ↔ Types** | `package-athler-js/src/display/` | schema 파일이 types를 import하고, types가 다시 schema를 import하는 양방향 의존 |
| **User Auth ↔ Axios** | `package-admin/src/api/` | 토큰 갱신 로직이 axios 인스턴스를 거쳐 다시 자기 자신을 호출하는 순환 체인 |

순환 참조는 **모듈 초기화 순서 오류**, **undefined import**, **번들 사이즈 증가**, **런타임 데드락** 등을 유발할 수 있어 제거가 필수적이었습니다.

---

## 2. 순환 참조 체인 상세 분석

### 2-1. Display Schema ↔ Types 순환참조

#### 변경 전 (순환 발생)

```
┌─────────────────────────┐
│  scroll-schema.ts       │
│  import { D_ScrollType }│──────┐
│  from '../types'        │      │
└─────────────────────────┘      │
          ▲                      ▼
          │              ┌──────────────────┐
          │              │  types/index.ts  │
          │              │  export * from   │
          │              │  './scroll'      │
          │              └────────┬─────────┘
          │                       │
          │                       ▼
          │              ┌──────────────────────────────┐
          │              │  types/scroll.ts              │
          └──────────────│  import { D_ScrollUITypeSchema│
                         │  } from '../schema/scroll-    │
                         │  schema'                      │
                         └──────────────────────────────┘
```

**순환 체인**:
```
scroll-schema.ts → types/index.ts → types/scroll.ts → scroll-schema.ts
                                                        ↑ 다시 처음으로
```

이 순환은 `types/index.ts`가 barrel export(`export *`)로 모든 타입 파일을 재수출하면서, schema에서 types를 import하면 **types 하위의 모든 파일**이 로드되고, 그 중 `scroll.ts`가 다시 `scroll-schema.ts`를 import하여 발생했습니다.

이 **하나의 순환 고리**가 barrel export를 통해 **연쇄적으로 22개의 순환 경로**를 만들어냈습니다.

#### 변경 후 (순환 해소)

```
┌──────────────────────────────────┐
│  scroll-schema.ts                │
│                                  │
│  const UI_TYPES = [...] as const │
│  export const D_ScrollUITypeSchema│
│    = z.enum(UI_TYPES)            │
│                                  │
│  // 로컬에서 직접 타입 정의       │
│  type D_ScrollType               │  ← import 제거, 자체 추론
│    = z.infer<typeof              │
│      D_ScrollUITypeSchema>       │
└──────────────────────────────────┘
          ▲ (단방향만 존재)
          │
┌──────────────────────────────┐
│  types/scroll.ts              │
│  import { D_ScrollUITypeSchema│
│  } from '../schema/scroll-    │
│  schema'                      │
└──────────────────────────────┘
```

**해결 방법**: `scroll-schema.ts`가 `../types`를 import하는 대신, `z.infer<typeof D_ScrollUITypeSchema>`로 **로컬에서 타입을 직접 정의**. 기존 `banner-schema.ts`에서 사용 중이던 패턴을 따른 것입니다.

---

### 2-2. User Auth ↔ Axios 순환참조

#### 변경 전 (순환 발생)

```
┌───────────────────┐       ┌────────────────┐       ┌─────────────┐       ┌───────────┐
│ axios-instance.ts │──────▶│ refresh-auth.ts│──────▶│ user-auth.ts│──────▶│ fetcher.ts│
└─────────▲─────────┘       └────────────────┘       └─────────────┘       └─────┬─────┘
          │                                                                       │
          └───────────────────────────────────────────────────────────────────────┘
                                    순환 고리 완성
```

**순환 체인**:
```
axios-instance.ts → refresh-auth.ts → user-auth.ts → fetcher.ts → axios-instance.ts
                                                                    ↑ 다시 처음으로
```

이 순환은 단순히 import 순서 문제를 넘어, **401 인터셉터로 인한 런타임 데드락** 위험도 내포하고 있었습니다:
- 401 에러 발생 시 `refreshAuth()` → `UserAuthAPI.getAccessToken()` 호출
- `fetcherWithoutToken`이 **동일한 `axiosInstance`**(401 인터셉터 포함)를 사용
- 토큰 갱신 요청이 다시 401을 받으면 무한 루프 가능성

#### 변경 후 (순환 해소)

```
┌──────────────────────┐
│   axios-config.ts    │ ◀── 새로 생성 (순수 설정, 의존성 0)
│   axiosBaseConfig    │
└──────┬───────┬───────┘
       │       │
       ▼       ▼
┌──────────┐  ┌──────────────────┐       ┌──────────────────┐
│ axios-   │  │  user-auth.ts    │◀──────│  refresh-auth.ts │
│ instance │  │                  │       │                   │
│ .ts      │  │ axiosRefresh     │       │ import {UserAuth  │
│          │──▶│ Instance =      │       │ API}              │
│ (인터셉터│  │ axios.create(    │       └──────────────────┘
│  포함)   │  │ axiosBaseConfig) │
└──────────┘  │                  │
              │ (인터셉터 없음,  │
              │  데드락 방지)    │
              └──────────────────┘
```

**해결 방법**:
1. **`axios-config.ts` 신규 생성** — axios 공통 설정을 별도 파일로 분리 (의존성 0개인 leaf 모듈)
2. **`user-auth.ts`에서 `fetcher.ts` import 제거** — `fetcherWithoutToken` 대신 **전용 인스턴스** `axiosRefreshInstance` 직접 생성
3. **`fetcher.ts`에서 `fetcherWithoutToken` 함수 제거** — 더 이상 사용처 없음

---

## 3. 수치적 결과

> `npx madge --circular --extensions ts,tsx` 기준 측정

### 3-1. Display Schema 영역 (`package-athler-js/src/display/schema/`)

| 측정 항목 | 변경 전 (develop) | 변경 후 (브랜치) | 변화 |
|-----------|:-----------------:|:----------------:|:----:|
| **순환 참조 수** | **22개** | **0개** | **-22개 (100% 제거)** |

<details>
<summary>변경 전 22개 순환참조 상세 목록 (펼치기)</summary>

```
 1) types/entity/order.ts > types/entity/payment.ts
 2) types/entity/product.ts > types/entity/review.ts > types/entity/order.ts
 3) types/entity/review.ts > types/entity/order.ts
 4) types/banner.ts > types/display.ts
 5) types/display.ts > types/custom.ts
 6) types/display.ts > types/margin.ts
 7) types/display.ts > types/product-query.ts
 8) types/display.ts > types/product-util.ts
 9) types/banner.ts > types/display.ts > types/product-util.ts > schema/index.ts
   > schema/display.ts > scroll-schema.ts > types/index.ts
10) schema/index.ts > schema/display.ts > scroll-schema.ts > types/index.ts > types/brand.ts
11) types/display.ts > types/product-util.ts > schema/index.ts > schema/display.ts
   > scroll-schema.ts > types/index.ts
12) types/display.ts > types/product-util.ts > schema/index.ts > schema/display.ts
   > scroll-schema.ts > types/index.ts > types/etc.ts
13) schema/index.ts > schema/display.ts > scroll-schema.ts > types/index.ts > types/etc.ts
14) types/product-util.ts > schema/index.ts > schema/display.ts > scroll-schema.ts
   > types/index.ts
15) types/display.ts > types/product-util.ts > schema/index.ts > schema/display.ts
   > scroll-schema.ts > types/index.ts > types/product.ts
16) types/product-util.ts > schema/index.ts > schema/display.ts > scroll-schema.ts
   > types/index.ts > types/product.ts
17) types/display.ts > types/product-util.ts > schema/index.ts > schema/display.ts
   > scroll-schema.ts > types/index.ts > types/product.ts > types/skeleton.ts
18) types/display.ts > types/product-util.ts > schema/index.ts > schema/display.ts
   > scroll-schema.ts > types/index.ts > types/scroll.ts
19) scroll-schema.ts > types/index.ts > types/scroll.ts
20) types/display.ts > types/product-util.ts > schema/index.ts > schema/display.ts
   > scroll-schema.ts > types/index.ts > types/standalone-filter.ts
21) types/display.ts > types/product-util.ts > schema/index.ts > schema/display.ts
   > scroll-schema.ts > types/index.ts > types/title.ts
22) types/banner.ts > types/display.ts > util/get-events.ts
```

</details>

**핵심 원인**: `scroll-schema.ts`에서 `../types`로의 import **1건**이 barrel export(`types/index.ts`)를 통해 연쇄적으로 22개의 순환 경로를 생성.

### 3-2. Admin API 영역 (`package-admin/src/api/`)

| 측정 항목 | 변경 전 (develop) | 변경 후 (브랜치) | 변화 |
|-----------|:-----------------:|:----------------:|:----:|
| **순환 참조 수** | **4개** | **3개** | **-1개** |
| **제거된 순환** | `refresh-auth.ts → user-auth.ts → fetcher.ts → axios-instance.ts` | - | **해결 완료** |

> 잔여 3개(`types/entity/` 내 order↔payment, product↔review↔order 순환)는 공통 타입 패키지의 순환이며 이 브랜치의 범위 밖입니다.

### 3-3. 종합 결과

```
┌────────────────────┬───────────┬───────────┬──────────────────┐
│     영역           │ 변경 전   │ 변경 후   │ 감소             │
├────────────────────┼───────────┼───────────┼──────────────────┤
│ Display Schema     │   22개    │    0개    │ -22개 (100%)     │
│ Admin API          │    4개    │    3개    │  -1개            │
├────────────────────┼───────────┼───────────┼──────────────────┤
│ 합계               │   26개    │    3개    │ -23개 (88% 감소) │
└────────────────────┴───────────┴───────────┴──────────────────┘
```

| 지표 | 수치 |
|------|------|
| **제거된 순환참조** | **26개 → 3개 (23개 제거, 88% 감소)** |
| **이 브랜치 대상 순환참조** | **23개 → 0개 (100% 해소)** |
| **변경된 파일 수 (패키지 코드)** | 11개 |
| **신규 생성** | 1개 (`axios-config.ts`) |
| **삭제** | 1개 (`get-custom-header-h.ts`) |

> 잔여 3개는 `types/entity/` 패키지의 entity 간 상호참조로, 이 브랜치의 대상이 아닙니다.

---

## 4. 변경 파일 요약

| 파일 | 변경 유형 | 설명 |
|------|-----------|------|
| `package-admin/src/api/axios-config.ts` | **신규** | axios 공통 설정 분리 (순환 차단용 leaf 모듈) |
| `package-admin/src/api/user-auth.ts` | 수정 | `fetcherWithoutToken` → 전용 `axiosRefreshInstance` 직접 사용 |
| `package-admin/src/api/fetcher.ts` | 수정 | `fetcherWithoutToken` 함수 제거 |
| `package-admin/src/api/axios-instance.ts` | 수정 | 공통 설정을 `axiosBaseConfig`에서 import |
| `package-athler-js/src/display/schema/scroll-schema.ts` | 수정 | `D_ScrollType` import 제거, 로컬 `z.infer` 정의로 대체 |
| `package-athler-js/src/display/hooks/use-auto-scroll-to-display.ts` | 수정 | `D_Data` import 및 `contents` 의존성 제거 |
| `package-athler-js/src/display/hooks/use-dp-context.tsx` | 수정 | `useAutoScrollToDisplay`에서 `contents` prop 제거 |
| `wish-list/brand/brand-wish-list-route.tsx` | 수정 | `getCustomHeaderH` → 인라인 로직 |
| `wish-list/cody/cody-wish-list-route.tsx` | 수정 | `getCustomHeaderH` → 인라인 로직 |
| `wish-list/product/product-wish-list-route.tsx` | 수정 | `getCustomHeaderH` → 인라인 로직 |
| `wish-list/__util__/get-custom-header-h.ts` | **삭제** | 불필요 유틸 파일 제거 |

---

## 5. 적용된 해결 패턴

| 패턴 | 적용 위치 | 설명 |
|------|-----------|------|
| **Local Type Inference** | `scroll-schema.ts` | 외부 types import 대신 `z.infer<typeof Schema>`로 로컬 정의 |
| **Leaf Module Extraction** | `axios-config.ts` | 의존성이 없는 순수 설정 모듈을 분리하여 순환 체인 차단 |
| **Dedicated Instance** | `user-auth.ts` | 공유 인스턴스 대신 전용 인스턴스를 생성하여 의존 경로 분리 |
| **Dead Code Removal** | `fetcher.ts`, `get-custom-header-h.ts` | 순환 원인이 되는 불필요 코드 제거 |
