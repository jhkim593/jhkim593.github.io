---
layout: post
title: "Junit으로 WebSocket [Stomp] Test"
author: "jhkim593"
tags: SpringBoot

---

## WebSocket
WebSocket은 기존의 단방향 HTTP프로톨과 호환되어 양방향 통신을 제공하기 위해 개발된 프로토콜입니다. 일반 Socket통신과 달리 Http 80 port를 이용하므로 방화벽에 제약이 없으며 통산 WebSocket으로 불립니다.

## Stomp
Stomp는 메시징 전송을 효율적으로 하기 위해 나온 프로토콜 이며 기본적으로 pub/sub 구조로 되어있어 메시지를 발송하고, 메시지를 받아 처리하는 부분이 정해져 있기 때문에 명확하게 인지하고 개발할 수있는 이점이 있습니다. 또한 Stomp를 이용하면 통신 메시지 헤더에 값을 세팅할 수있어 헤더 값을 기반으로 통신 시 인증 처리를 구현하는 것도 가능합니다. 위에서 언급한 pub/sub란 메세지를 공급하는 주체와 소비하는 주체를 분리해 제공하는 메시징 방법입니다.
- 채팅방 생성 : pub/sub 구현을 위한 Topic이 생성됨
- 채팅방 입장 : Topic 구독
- 채팅방에서 메시지를 송수신 : 해당 Topic으로 메세지를 송신(pub), 메세지를 수신(sub)


~~~gradle
implementation 'org.springframework.boot:spring-boot-starter-websocket'
implementation 'org.webjars:stomp-websocket:2.3.3-1'
implementation 'org.webjars:sockjs-client:1.1.2'
~~~

<br>

~~~java
@Configuration
@EnableWebSocketMessageBroker
@RequiredArgsConstructor
public class WebSocketConfig implements WebSocketMessageBrokerConfigurer {

    private final StompConfig stompConfig;

    @Override
    public void configureMessageBroker(MessageBrokerRegistry config) {
        config.enableSimpleBroker("/topic","/queue");
        config.setApplicationDestinationPrefixes("/");
    }

    @Override
    public void registerStompEndpoints(StompEndpointRegistry registry) {
        registry.addEndpoint("/ws").setAllowedOrigins("*")
                .withSockJS();

    }

    @Override
    public void configureClientInboundChannel(ChannelRegistration registration) {
        registration.interceptors(stompConfig);
    }

}
~~~
- @EnableWebSocketMessageBroker를 통해 메시지를 모으기 위한 설정
- `/`로 시작되는 stomp message는 @Controller 내부에 @MessageMapping 메소드로 라우팅 됩니다.
- `/topic`, `/queue`로 시작하는 메시지를 브로커로 라우팅
- configureClientInboundChannel를 통해 ChannelInterceptor를 설정합니다.

<br>

~~~java

@Component
@RequiredArgsConstructor

public class StompConfig implements ChannelInterceptor {

    private final JwtTokenProvider jwtTokenProvider;
    private final UserRepository userRepository;
    private final ChatMessageRepository chatMessageRepository;

    @Override
    public Message<?> preSend(Message<?> message, MessageChannel channel) {

        StompHeaderAccessor accessor = MessageHeaderAccessor.getAccessor(message, StompHeaderAccessor.class);
        if (StompCommand.CONNECT == accessor.getCommand()) {
            String token = accessor.getFirstNativeHeader("Authorization");

            if (hasText(token) && token.startsWith("Bearer")) {
                token=token.replace("Bearer ","");
            }
            else{
                return null;
            }

            Long userId = Long.valueOf(jwtTokenProvider.getUserPk(token));

            User user = userRepository.findById(userId).orElseThrow(UserNotFoundException::new);

            accessor.setUser(jwtTokenProvider.getAuthentication(token));

            chatMessageRepository.createRoomSession(user.getEmail());

        }
        else if(StompCommand.DISCONNECT == accessor.getCommand()){
            if (accessor.getUser()!=null){
                chatMessageRepository.deleteRoomSession(accessor.getUser().getName());
            }
        }
        return message;
    }

}
~~~



ChannelInterceptor를 통해 소켓 통신 연결 , 해제 감지 및 헤더 값에 회원 정보가 포함 된 정보를 이용해 인증을 처리할 수있습니다. 소켓 연결시 토큰을 이용해 인증된 회원 일 경우에만 성공 하도록 설정했습니다.

<br>

~~~java
@RequiredArgsConstructor
@Controller
public class ChatController {

    @MessageMapping("/message")
    public void message(@RequestBody @Valid ChatMessageDto message) {

      messagingTemplate.convertAndSend("/queue/chat/" + message.getUserId(), roomMessage);

    }
}
~~~

- @MessageMapping
  목적지를 기반으로 메시지를 라우팅하는 메소드에 annotation을 달 수있습니다.
- @SendTo , @SendToUser 사용여 message목적지를 커스터마이징 할 수 있지만
  SimpleMessagingTemplate를 사용하는 것과 다르지 않습니다.

<br>

~~~java

@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
public class StompTest {

   @LocalServerPort Integer port;

   BlockingQueue<String> blockingQueue;

   WebSocketStompClient stompClient;

   static final String WEBSOCKET_TOPIC = "/queue/chat/"

   @Autowired ObjectMapper mapper;

   @Autowired SignService signService;


   @Test
   public void sendChatMessage() throws Exception {
       blockingQueue = new LinkedBlockingDeque<>();
       stompClient = new WebSocketStompClient(new SockJsClient(
               Arrays.asList(new WebSocketTransport(new StandardWebSocketClient()))));

       UserLoginResponseDto basic = signService.login(new UserLoginRequestDto("fpdlwjzlr@naver.com", "11"));

       StompHeaders headers = new StompHeaders();
       headers.add("Authorization", "Bearer " +basic.getToken().getAccessToken());
       StompSession session = stompClient
               .connect(getWsPath(), new WebSocketHttpHeaders(), headers, new StompSessionHandlerAdapter() {
               })
               .get(5, SECONDS);

       session.subscribe(WEBSOCKET_TOPIC + basic2.getProfile().getId(), new DefaultStompFrameHandler());
       ChatMessageDto chatMessageDto = new ChatMessageDto(MessageType.TALK, 1L, basic.getProfile().getId(), basic.getProfile().getName(), "message", null, LocalDateTime.now().withNano(0), false);

       session.send("/message", mapper.writeValueAsString(chatMessageDto).getBytes(StandardCharsets.UTF_8));

       // then
       String jsonResult = blockingQueue.poll(5, SECONDS);

       session.disconnect();

       Map<String, String> result = gson.fromJson(jsonResult, new HashMap().getClass());
       JSONParser jsonParser = new JSONParser();
       JSONObject jsonObject = (JSONObject) jsonParser.parse(jsonResult);

       assertThat(result.get("message")).isEqualTo(chatMessageDto.getMessage());

   }
   private String getWsPath() {
       return String.format("ws://localhost:%d/ws", port);
   }

   class DefaultStompFrameHandler implements StompFrameHandler {
       @Override
       public Type getPayloadType(StompHeaders stompHeaders) {
           return byte[].class;
       }

       @Override
       public void handleFrame(StompHeaders stompHeaders, Object o) {
           blockingQueue.offer(new String((byte[]) o));
       }
   }
}
~~~
session.send(/message, ...)명령어를 통해 컨트롤러 내부에 @MessageMapping("/message")이 선언된 메소드로 라우팅 됩니다.
StompFrameHandler에 getPayloadType을 통해 변환 할 Object 유형을 결정하고 데이터를 처리하게 되며 handleFrame메소드를 통해 전송된 데이터를 blockingQueue에 저장합니다.
