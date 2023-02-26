---
title: WebSocket - 멀티스레드 환경에서 사용시 synchronized 키워드 없애기
published: false
tags: websocket, spring
---

웹소켓(WebSocket)은 하나의 TCP 커넥션을 통해서 클라이언트와 서버간의 양방향 통신을 지속하게 해주는 프로토콜입니다.
제가 실무에서 사용한 용례는 특정 상품의 가격 정보를 나타내는 챠트를 만드는 경우였습니다.

종목의 실시간 정보 및 챠트 데이터 혹은 다른 사례로 채팅의 대화글 등, 
클라이언트가 명시적으로 요청하지 않아도 서버에서 클라이언트의 특정 데이터 상태를 갱신해야만 하는 상황에서 유용한 것 같습니다.

---

# 1. Spring Boot 에서 WebSocket 사용하기

```xml
<dependencies>
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-websocket</artifactId>
    </dependency>
</dependencies>
```
스프링부트에서 웹소켓을 사용하는 간단한 방법은 위와 같이 스프링부트 스타터 의존성을 사용하는 것입니다.

```java
@Component
public class WebSocketBroadcaster extends TextWebSocketHandler {

  private final Map<String, WebSocketSession> sessionMap = new ConcurrentHashMap<>();

  public void broadcast(String message) {
    sessionMap.values()
        .forEach(session -> {
          try {
            session.sendMessage(new TextMessage(message));
          } catch (IOException e) {
            e.printStackTrace();
          }
        });
  }

  @Override
  public void afterConnectionEstablished(WebSocketSession session) {
    sessionMap.put(session.getId(), session);
    session.sendMessage(new TextMessage("connected"));
  }

  @Override
  protected void handleTextMessage(WebSocketSession session, TextMessage message) {
    System.out.println(message);
  }

  @Override
  public void afterConnectionClosed(WebSocketSession session, CloseStatus status) {
    try (session) {
      sessionMap.remove(session.getId());
    }
  }
}

```

의존성을 추가한 뒤, 웹소켓 서버 구현을 위해 제공해주는 템플릿 메소드 클래스를 확장합니다. 예제에서는 'TextWebSocketHandler' 타입을 사용했습니다.
이외에도 'BinaryWebSocketHandler' 타입도 있습니다. 패킷 크기를 줄이기 위해 독자적인 인코딩으로 직렬화한다면 이 바이너리 클래스를 확장하면 됩니다.

`afterConnectionEstablished` 메소드을 오버라이딩하여 연결된 클라이언트의 세션을 `ConcurrentHashMap` 컨테이너에 저장합니다.
데모가 아닌 실제 사용시에는 이 세션객체를 구성요소로 하는 추가정보를 포함하는 랩퍼 클래스를 사용할 수도 있겠습니다.

`afterConnectionClosed`메소드를 오버라이딩하여 웹소켓 커넥션이 종료된 경우 수행할 훅을 정의합니다.
세션객체는 `Closeable` 을 구현하고 있으므로 `try-with-resources`을 사용하면 편리합니다.

마지막으로 연결된 모든 세션들에게 데이터를 전달하는 `broadcast` 메소드를 정의했습니다.

```java
@RequiredArgsConstructor
@EnableWebSocket
@Configuration
public class WebsocketConfig implements WebSocketConfigurer {

  private final WebSocketHandler webSocketHandler;

  @Override
  public void registerWebSocketHandlers(WebSocketHandlerRegistry registry) {
    registry.addHandler(webSocketHandler, "/bin");
  }
}

```

작성한 웹소켓 핸들러를 설정 빈 클래스에서 등록합니다. 경로를 다르게 하여 여러개의 핸들러를 등록할 수 있습니다.

---

# 2. 메시지 전달

## 2.1 싱글스레드인 경우
```java
@RequiredArgsConstructor
@RestController
public class WebSocketController {

  private final WebSocketBroadcaster webSocketBroadcaster;

  @PostMapping("/ws/test")
  public ResponseEntity<?> test(@RequestBody String message) {
    webSocketBroadcaster.broadcast(message);
    return ResponseEntity.ok().build();
  }
}

```

