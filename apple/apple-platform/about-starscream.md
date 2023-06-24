---
description: https://github.com/daltoniam/Starscream
---

# Starscream 에 대하여

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

WalletConnect 의 Example 앱은 Starscream 3.x.x 버전을 사용하고 있더라구요?? 아니 Starscream 이 4.x.x 버전이 훨씬 이전에 만들어 졌는데 왜 구버전을 쓰고있지 싶어서 소스를 봤더니.. 3.x.x 버전은 swift 파일이 4개 밖에 없더라구요?

### 모르겠고 일단 downgrade 하고 돌려보자.

일단 되긴된다! 앱이 Background 상태로 진입해도 socket 을 종료하지 않는다! (유레카)

근데 문제는 `protocol` 구조가 많이 달랐어요.

저는 4.0.4 기준으로 코드를 작성 해놨거든요. 3.x.x 버전을 사용하려면 코드를 수정해야 했죠. 4.0.4 에서는 `WebSocketEvent` 가 `enum` 으로 되어 있어서 3.x.x 에 있는 protocol method 들이 하나의 method `didReceive(event:client:)` 에 있는 거죠. (난 이게 더 보기 불편한 것 같은데.. switch case 도 남발하기 싫고...)

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

event 의 case 마다 일일히 옮겨도 되지만... 그거알죠? 옮겼는데 생각하고 다르게 동작하면 코드 되돌려야할 수도 있는거 ㅎㅎ

그래서 4.0.4 버전에서 뭐가 달라졌고, 3.x.x 버전에서는 왜 Background 이슈가 없는지 찾아봤어요. Starscream 소스를요..

## 결론

Starscream 4.0.4 버전에서는 WebSocket 객체 초기화 시 engine 을 설정할 수 있는데 설정하지 않으면 OS 버전에 맞게 적절한 engine 을 사용함. (iOS 16.4 시뮬레이터를 이용했기 때문에 `NativeEngine` 을 engine 으로 주입받았던 것.

<pre class="language-swift" data-title="WebSocket.swift" data-full-width="false"><code class="lang-swift"><strong>public convenience init(request: URLRequest, certPinner: CertificatePinning? = FoundationSecurity(), compressionHandler: CompressionHandler? = nil, useCustomEngine: Bool = true) {
</strong>    if #available(macOS 10.15, iOS 13.0, watchOS 6.0, tvOS 13.0, *), !useCustomEngine {
        self.init(request: request, engine: NativeEngine())
    } else if #available(macOS 10.14, iOS 12.0, watchOS 5.0, tvOS 12.0, *) {
        self.init(request: request, engine: WSEngine(transport: TCPTransport(), certPinner: certPinner, compressionHandler: compressionHandler))
    } else {
        self.init(request: request, engine: WSEngine(transport: FoundationTransport(), certPinner: certPinner, compressionHandler: compressionHandler))
    }
}
</code></pre>

### 그래서 `NativeEngine` 은?&#x20;

Foundation Framework에 포함된 `URLSessionWebSocketTask` 을 사용해서 소켓 통신을 하고 있어요. 비교적 친숙하네요. HTTP 통신을 위해 URLSession을 자주 사용하잖아요?? 사촌지간 같은 거죠. `URLSessionDataDelegate`, `URLSessionWebSocketDelegate` 두개의 Delegate도 구현이 되어있네요. 정확한 것은 더 알아봐야 하겠지만, `URLSessionWebSocketTask` 는 앱이 Background 상태로 진입할 때 socket 연결을 종료시키고 있었습니다.

### FoundationTransport

Starscream 3.x.x 버전에서 쓰던 코드와 동일한 녀석들이 포함되어 있는 객체였어요. `CoreFoundation` Framework 안에 [CFStream](https://developer.apple.com/documentation/corefoundation/cfstream) 이라는 녀석을 사용해서 socket 통신을 구현해 놓았어요. 와.. 이런 소스는 처음 봤어요. [`CFStreamCreatePairWithSocketToHost(::::)`](https://developer.apple.com/documentation/corefoundation/1539739-cfstreamcreatepairwithsockettoho) 딱 봐도 진즉에 Deprecated 되었을 것 같았거든요? 근데 놀랍게도 iOS15 이후부터 Deprecated되었네요.&#x20;

{% code title="func connect(::::) 중" %}
```swift
var readStream: Unmanaged<CFReadStream>?
var writeStream: Unmanaged<CFWriteStream>?
let h = url.host! as NSString
CFStreamCreatePairWithSocketToHost(nil, h, UInt32(port), &readStream, &writeStream)
inputStream = readStream!.takeRetainedValue()
outputStream = writeStream!.takeRetainedValue()
```
{% endcode %}

### 내 소스

```swift
// 기존 (Starscream 내부에서 URLSessionWebSocketTask 를 사용)
let socket = WebSocket(request: request)

// 변경 (engine 을 주입함. CFStream 사용)
let engine = WSEngine(transport: FoundationTransport(), certPinner: FoundationSecurity())
let socket = WebSocket(request: request, engine: engine)
```

2 line 변경으로 다른 코드는 거의 건드리지 않고 Background 에서 socket 을 끊지 않도록 했어요. (ping / pong 시 timeout 이 발생하는 경우는 socket을 끊더라구요. 사실 이건 당연한 결과지요.)





공유할 생각이 있다면 언제든지 편하게 연락주세요.



Author [JiHoon Lee](http://localhost:5000/u/f7NfZX9yQTNV6NucvAP7UGpv3213 "mention")

Email [**dev@2rick.com**](mailto:dev@2rick.com)
