# JS Sentry Fatal 스터디

JS Fatal 에러와 Sentry 모니터링에 대한 학습 노트.

이를 통해서 아래의 대시보드를 구성했다.

https://bind-vw.sentry.io/dashboard/129466/

![[README-1770471030451.webp]]

## 문서 목록

| 문서 | 내용 |
|------|------|
| [error_handled_false와 onunhandledrejection란](error_handled_false와_onunhandledrejection란.md) | `error.handled:false`와 `level:fatal`의 차이, 비유를 통한 개념 설명 |
| [Fatal에러와 window error onunhandledrejection는 무엇인가](그래서_Fatal에러와_window_error_onunhandledrejection는_무엇인가.md) | SDK의 에러 수집 원리, Fatal 에러의 기초 개념 |
| [Fatal Promise 대시보드 세팅법](Fatal_Promise_대시보드_세팅법.md) | Sentry 대시보드에서 Fatal/Promise 에러 필터링 및 위젯 세팅 방법 |
