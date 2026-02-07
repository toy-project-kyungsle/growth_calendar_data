
2026-02-07

현재 코드에서 **둘 다 제대로 Sentry에 보내지고 있습니다.** 대시보드에서 보는 방법을 설명하겠습니다.

---

## JS Fatal 발생률 (목표: <10건/10k 세션)

### Sentry Issues에서 필터링

```
is:unresolved level:fatal
```

JS Fatal은 SDK가 자동으로 `level: fatal`을 붙여서 보내므로, Issues 페이지에서 바로 필터 가능합니다.

### Discover 쿼리 (위젯용)

```yaml
Search: event.type:error level:fatal
Visualize: count()
Group by: (없음)
Display: Big Number
```

"10k 세션 당" 비율을 보려면:

```yaml
# 위젯 1: Fatal 건수
Search: event.type:error level:fatal
Visualize: count()

# 위젯 2: 세션 수
# → Release Health 위젯에서 "Total Sessions" 사용
# → 또는 Sessions 데이터셋에서 count_unique(session)

# 비율 = 위젯1 / 위젯2 * 10000
```

### 더 정확하게: Unhandled JS Error만 보기

```yaml
Search: event.type:error error.handled:false
Visualize: count()
```

`error.handled:false`는 SDK가 글로벌 에러 핸들러에서 잡은 것 = 코드에서 try/catch 안 한 진짜 Fatal입니다.

---

## Unhandled Promise Rejection (목표: <5건/10k 세션)

### Sentry Issues에서 필터링

```
is:unresolved mechanism:onunhandledrejection
```

또는 검색창에:

```
error.mechanism:onunhandledrejection
```

### Discover 쿼리 (위젯용)

```yaml
Search: event.type:error error.mechanism:onunhandledrejection
Visualize: count()
Group by: (없음)
Display: Big Number
```

시간 추이를 보려면:

```yaml
Search: event.type:error error.mechanism:onunhandledrejection
Visualize: count()
Display: Line Chart
Interval: 1h (또는 1d)
```

---

## 대시보드 위젯 구성 예시

### 위젯 1: JS Fatal 발생률 (Big Number)

|설정|값|
|---|---|
|Dataset|**Errors (Events)**|
|Search|`event.type:error error.handled:false`|
|Visualize|`count()`|
|Display|**Big Number**|
|Period|Last 24 hours|

### 위젯 2: Unhandled Promise Rejection (Big Number)

|설정|값|
|---|---|
|Dataset|**Errors (Events)**|
|Search|`event.type:error error.mechanism:onunhandledrejection`|
|Visualize|`count()`|
|Display|**Big Number**|
|Period|Last 24 hours|

### 위젯 3: 릴리즈별 안정성 추이 (Line Chart)

|설정|값|
|---|---|
|Dataset|**Errors (Events)**|
|Search|`event.type:error error.handled:false`|
|Visualize|`count()`|
|Group by|`release`|
|Display|**Line Chart**|

### 위젯 4: Top Unhandled Errors (Table)

|설정|값|
|---|---|
|Dataset|**Errors (Events)**|
|Search|`event.type:error error.handled:false`|
|Columns|`title`, `count()`, `count_unique(user)`|
|Sort|`count()` desc|
|Display|**Table**|

---

## Sentry Alerts 설정 (임계값 도달 시 알림)

### JS Fatal > 10건/시간 알림

```
Metric Alert:
  When: count(event.type:error error.handled:false) > 10
  Time window: 1 hour
  Action: Slack / Email
```

### Unhandled Promise Rejection > 5건/시간 알림

```
Metric Alert:
  When: count(event.type:error error.mechanism:onunhandledrejection) > 5
  Time window: 1 hour
  Action: Slack / Email
```

---

## 핵심 필터 키 정리

|필터|의미|용도|
|---|---|---|
|`error.handled:false`|SDK가 자동 캡처한 에러 (try/catch 안 된 것)|JS Fatal|
|`error.handled:true`|`captureException()`으로 명시적 전송한 에러|Handled 에러|
|`error.mechanism:onunhandledrejection`|Promise rejection|Unhandled Promise|
|`error.mechanism:onerror`|글로벌 JS 에러|JS 크래시|
|`level:fatal`|Fatal 레벨 에러|앱 크래시급|
|`level:error`|Error 레벨|일반 에러|

`error.handled:false`가 가장 핵심입니다. 이것 하나로 "사용자가 실제로 맞닥뜨린 처리 안 된 에러"를 정확히 볼 수 있습니다.