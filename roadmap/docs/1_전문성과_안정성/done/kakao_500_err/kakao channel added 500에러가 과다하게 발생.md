  
# 1. 발견
카카오 채널 추가 확인 API(`KakaoChannelAddedViewSet`)에서 500 에러가 과도하게 발생하고 있었다.
프론트에서는 500 에러에 대해서는 모두 프론트엔드 센트리로 보내주고 있다.
센트리 로그가 대량으로 쌓이면서 실제 서버 장애와 사용자 측 문제를 구분하기 어려운 상황이었다.
  
500 에러가 이렇게 많이 발생하는 이유를 자세하게 알아보지는 않았다. 서버 코드를 확인하던 중 카카오 API 호출 시 사용자 계정 문제(비활성, 미연결, 미등록)로 인한 에러가 모두 500으로 반환되고 있음을 발견했다.
이 에러들은 400으로 보내주어도 될 것이라 판단하여 서버 코드 수정을 진행했다.
  
# 2. 변경 내용
## 2-1. 커스텀 예외 추가 (`users/exceptions.py`)
`KakaoAccountNotAvailableError` 예외 클래스를 신규 생성했다.

|항목|값|
|---|---|
|status_code|400|
|default_detail|"카카오톡 계정 상태를 확인할 수 없습니다. 카카오톡 연동 상태를 확인해주세요."|
|default_code|"kakao_account_not_available"|

이는 카카오 계정에 문제가 있는 유저가 요청할 시 사용하는 클래스이다.
status code로 인해서 이 에러는 400의 상태로 프론트엔드에 전달될 것이다.
  
## 2-2. 에러 분기 처리 (`users/views/kakao_channel_added_view.py`)
카카오 API 에러 응답에서 아래 3가지 경우를 400(`KakaoAccountNotAvailableError`)으로 분리했다.
- `"the account is inactive."` - 계정 비활성
- `"given account is not connected to any talk user."` - 카카오톡 미연결
- `"NotRegisteredUserException"` - 미등록 사용자
나머지 에러는 기존대로 `APIException` (500) 으로 처리한다.
  
만약 이렇게 했는데 500에러의 수치가 떨어지지 않는다면 다른 방법을 고안한다.
  
# 3. 결과
  
결과를 굳이 %로 나누지 않아도 될 정도로 대부분의 500에러가 사라졌다.
그런데.. 이렇게까지 많이 사라진다고 하면 대부분 에러가 `2-2` 에서의 에러라는 말인데,
그것도 나름 큰 일이긴 하다. 저 에러들에 대해서 더 깊게 파봐야 할 듯하다.
  
[[세 가지 에러들에 대한 정의]]
  
이 에러들을 추적해보았을 때 아마
`전화번호로 회원가입한 유저이자, 카카오로 로그인을 한 적이 없는 유저`
한테 발생할 것이라고 생각했다.
  
그러나 `hasKakaoAccount=true` 로 user info 가 세팅되어 있는 유저에게서 400에러가 발생하는 것을 테스트 서버에서 확인했다. 사건이 미궁속으로 빠지고 있는데, 더 파보기에는 그렇게 중요한 에러는 아닌 것 같아 여기서 마무리한다. 센트리 에러를 줄이고자 하는 소기의 목표는 달성하였다.
  
## 3-1. 개선 전

> 26.02.10 00:00 ~ 26.02.13 13:00
  
[https://bind-vw.sentry.io/explore/discover/homepage/?dataset=errors&end=2026-02-13T04%3A00%3A59&field=title&field=project&field=user.display&field=timestamp&name=All Errors&query=%2Fv1%2Fuser%2Fkakao-channel-added&queryDataset=error-events&sort=-timestamp&start=2026-02-09T15%3A00%3A00&yAxis=count()](https://bind-vw.sentry.io/explore/discover/homepage/?dataset=errors&end=2026-02-13T04%3A00%3A59&field=title&field=project&field=user.display&field=timestamp&name=All%20Errors&query=%2Fv1%2Fuser%2Fkakao-channel-added&queryDataset=error-events&sort=-timestamp&start=2026-02-09T15%3A00%3A00&yAxis=count%28%29)
  
![[../../../../../../99.첨부파일/image 61.webp|image 61.png]]
  
- 총 개수 6,174
  
## 3-2. 개선 후

> 26.02.13 13:00 ~ 26.02.15 23:59
  
[https://bind-vw.sentry.io/explore/discover/homepage/?dataset=errors&end=2026-02-15T14%3A59%3A59&field=title&field=project&field=user.display&field=timestamp&name=All Errors&query=%2Fv1%2Fuser%2Fkakao-channel-added&queryDataset=error-events&sort=-timestamp&start=2026-02-13T05%3A00%3A00&yAxis=count()](https://bind-vw.sentry.io/explore/discover/homepage/?dataset=errors&end=2026-02-15T14%3A59%3A59&field=title&field=project&field=user.display&field=timestamp&name=All%20Errors&query=%2Fv1%2Fuser%2Fkakao-channel-added&queryDataset=error-events&sort=-timestamp&start=2026-02-13T05%3A00%3A00&yAxis=count%28%29)
  
![[../../../../../../99.첨부파일/image 1 39.webp|image 1 39.png]]
  
- 총 개수 98