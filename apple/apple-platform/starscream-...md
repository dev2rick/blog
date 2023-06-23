---
description: https://github.com/daltoniam/Starscream
---

# Starscream 에 대하여..

## TL;DR

1. [Starscream](https://github.com/daltoniam/Starscream) 은 Swift 로 작성된 Socket 통신을 위한 라이브러리이다.
2. socket.io swift 의 dependency 라이브러리.
3. 앱이 백그라운드로 진입시 socket disconnect 됨.
4. 최신버전인 4.x.x 이후에 발생하는 것으로 보임. (3.x.x 버전에서는 백그라운드 이슈가 없음)
5. 4.x.x 에서 백그라운드 이슈를 해결

## 서론

Client 애플리케이션을 개발하다보면 언젠가 사용하게 되는 Socket.io, Apple platform은 `Starscream`을 dependency로 포함하고 있어서 좋으나 싫으나 제품에 포함되게 됩니다.

### Starscream? 그걸 왜 단독으로 써? socket.io 쓰면 되는거 아니야??

socket.io 는 그들만의 규격이 있어서 서버에서 해당 규격을 지키지 않으면 소켓 통신을 할 수 없어요. (정확한건 아님) 제가 겪은 경우에는 해당규격없이 접속해서 통신해야 했어요. 그리고 socket.io 를 사용해야할 정도로 많은 기능을 구현할 필요가 없었어요.

어쨋든 socket.io 를 안쓰고 iOS 에서 소켓통신을 해야하는 미션이었어요.

### 개발 요구사항

조금 더 정확히는 \[App A] <-> \[Socket Server] <-> \[App B] 형태의 구조였지요. 소켓통신은 앱간의 정보전달을 위해 필요한 것이고, Socket Server 는 A 와 B 의 중계자 역할만 하고 있어요.

#### Flow

1. A앱이 B앱에 필요한 정보를 소켓에 요청을 하고 B앱을 Application scheme 을 이용하여 연다.
2. B앱은 소켓을 통해 요청된 정보를 받고, Response를 소켓에 전달한 뒤 A 앱을 scheme 을 통해 다시 열어줘요.

이 경우 두 앱(iOS 앱)은 Background / Foreground 상태전환을 계속 반복하게 되죠.

저는 A앱을 위한 `라이브러리를 개발`해야 했구요.

## 문제발생

Starscream 을 사용해서 통신성공, 필요한 기능 구현 완료.. 라고\~ 생각했던 찰라 A앱이 Background 진입시 소켓이 끊겨버리네?? 하.. 심각한 문제에 봉착했어요. Starscream 을 이용해서 개발 다 해놨더니 가장중요한게 안되는 거에요.

의존성도 강하게 결합되어 있어서 Starscream 을 제거하게 되면 대공사가 필요했어요. (머리가 나쁘면 몸이 고생한다고 했던가...)

잠시 심호흡하고 제가 하려던 것과 유사한 라이브러리인 [WalletConnectV2](https://github.com/WalletConnect/WalletConnectSwiftV2) 는 어떻게 하고 있나 찾아 봤어요. (WalletConnect 도 Starscream 을 사용하고 있는 것은 이미 조사를 통해 알고 있었거든요.)

## 단서 확보

WalletConnect 는 Starscream 3.x.x 버전을 사용하고 있더라구요?? 아니 Starscream 이 4.x.x 버전이 훨씬 이전에 만들어 졌는데 왜 구버전을 쓰고있지 싶어서 소스를 봤더니.. 3.x.x 버전은 swift 파일이 4개 밖에 없더라구요?

### 모르겠고 일단 downgrade 하고 돌려보자.

일단 되긴된다! 앱이 Background 상태로 진입해도 socket 을 종료하지 않는다! (유레카)

근데 문제는 `protocol` 구조가 많이 달랐어요.

저는 4.0.4 기준으로 코드를 작성 해놨거든요. 3.x.x 버전을 사용하려면 코드를 수정해야 했죠. 4.0.4 에서는 `WebSocketEvent` 가 `enum` 으로 되어 있어서 3.x.x 에 있는 protocol method 들이 하나의 method `didReceive(event:client:)` 에 있는 거죠. (난 이게 더 보기 불편한 것 같은데.. switch case 도 남발하기 싫고...)

일일히 옮겨도 되지만... 그거알죠? 옮겼는데 생각하고 다르게 동작하면 코드 되돌려야할 수도 있는거 ㅎㅎ

{% code title="Starscream 4.0.4" %}
```swift
public protocol WebSocketDelegate: class {
    func didReceive(event: WebSocketEvent, client: WebSocket)
}
```
{% endcode %}

{% code title="Starscream 3.1.2" %}
```swift
public protocol WebSocketDelegate: class {
    func websocketDidConnect(socket: WebSocketClient)
    func websocketDidDisconnect(socket: WebSocketClient, error: Error?)
    func websocketDidReceiveMessage(socket: WebSocketClient, text: String)
    func websocketDidReceiveData(socket: WebSocketClient, data: Data)
}
```
{% endcode %}

























