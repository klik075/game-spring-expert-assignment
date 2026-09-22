# WebCraft 실시간 게임 서버 구현
제공받은 웹 기반 Minecraft에 실시간 서버를 구현하는 프로젝트

발제 링크 : https://app.notion.com/p/teamsparta/260916-3dd2dc3ef514800592daddba671adc44

프로젝트 기간 : 2026.9.21(월) ~ 2026.9.22(화) 총 2일

## 1. 기술 스택
| 카테고리                 | 기술                       |
|--------------------------|----------------------------|
| **Language**             | Java 21                    |
| **Framework**            | Spring Boot 4.1.0          |
| **Web**                  | Spring MVC, WebSocket      |
| **ORM**                  | Spring Data JPA, Hibernate |
| **Infrastructure**       | Docker                     |
| **Database**             | MySQL 8.4                  |
| **In-Memory Data Store** | Redis 7                    |


## 2. API 명세

| 메소드 | 경로                        | 성공 | 하는 일          |
|--------|-----------------------------|-----:|------------------|
| POST   | `/players`                  |  201 | 플레이어 등록    |
| GET    | `/worlds`                   |  200 | 월드 목록 조회   |
| POST   | `/worlds`                   |  201 | 월드 생성        |
| GET    | `/worlds/{worldId}/chats`   |  200 | 최근 채팅 조회   |
| WS     | `/ws/worlds/{worldId}`      |  101 | WebSocket 연결   |

## 3. 오류 응답
| 에러                          |  상태 | 언제                                                   |
|-------------------------------|------:|--------------------------------------------------------|
| `VALIDATION_FAILED`           | `400` | 필수 값 누락, 길이·패턴 위반 또는 잘못된 숫자 파라미터 |
| `INVALID_REQUEST_BODY`        | `400` | 잘못된 JSON, 알 수 없는 필드 또는 난이도               |
| `WORLD_NOT_FOUND`             | `404` | 존재하지 않는 월드                                     |
| `PLAYER_NOT_FOUND`            | `404` | 등록되지 않은 닉네임으로 월드 생성                     |
| `DUPLICATE_NICKNAME`          | `409` | 이미 등록된 닉네임                                     |
| `WORLD_LIMIT_REACHED`         | `409` | 월드가 이미 3개인 상태에서 생성                        |
| `INTERNAL_ERROR`              | `500` | 예기치 못한 서버 오류                                  |
| `WORLD_BASELINE_INITIALIZING` | `503` | 서버 기동 직후 월드 준비가 끝나기 전                   |

## 4. ERD
1. players

<img src="/images/ERD_players.png" width="800">

2. worlds

<img src="/images/ERD_worlds.png" width="800">

3. chat_messages : worlds

<img src="/images/ERD_chatMessages_worlds.png" width="800">

---
## 5. 미션 (Lv 1 ~ 15)
### Lv 1. Docker로 MySQL과 Redis 설정
1. application.properties 작성

<img src="/images/lv1_properties.png" width="800">

2. docker-compose.yml 작성

<img src="/images/lv1_docker-compose.png" width="800">

3. 터미널로 `docker compose up -d` 실행

4. Docker로 MySQL, Redis 실행

<img src="/images/lv1_docker_containers.png" width="800">

---
### Lv 2. SQL을 JPA 인덱스로 표현하기
1. `@Table`의 indexes 속성을 사용하여 인덱스 정의
- `인덱스` : 데이터베이스에서 특정 데이터를 빠르게 찾기 위해 사용하는 자료구조다.
- `@Index`는 데이터베이스 테이블에 생성할 인덱스의 정보를 나타낸다.
- `@Index.name`은 생성할 인덱스 이름
- `@Index.columnList`는 인덱스를 생성할 테이블의 컬럼을 지정한다.
- `columnList = "A, B"`의 형태는 복합 인덱스가 만들어진다.
- `복합 인덱스` : 여러 컬럼을 묶어서 하나의 인덱스로 만든 것이다.<br>
컬럼 순서가 중요하고 왼쪽 컬럼을 정렬 후 다음 컬럼을 정렬한다.<br>
두 번째 컬럼만 조건으로 사용하는 조회는 적합하지 않다.<br>

<img src="/images/lv2_compositeKey.png" width="800">

---
### Lv 3. 요청 검증과 DTO: 플레이어 등록
1. nickname에 `@Pattern`으로 정규식 적용하기
- `정규식` : 문자열이 특정한 규칙을 만족하는지 표현하는 패턴이다.
- `^` : 문자열 시작
- `[]` : 문자 하나의 범위로<br>
  `a-z`      소문자 a ~ z<br>
  `A-Z`      대문자 A ~ Z<br>
  `0-9`      숫자 0 ~ 9<br>
- `{min, max}` : 반복 횟수로 `[min, max]`만큼 반복되어야 한다는 의미
- `$` : 문자열 끝
- `@Pattern`은 Jakarta Bean Validation에서 문자열이 특정 정규식과 일치하는지 검사하는 검증 어노테이션이다.
- `@Pattern.regex`는 검증에 사용할 정규식을 지정하는 속성

<img src="/images/lv3_CreatePlayerRequest.png" width="800">

2. PlayerController 설정 
- `@PostMapping`: 요청 URL 지정
- `@Valid`: 요청 데이터 검증
- `@RequestBody`: 요청 Body를 객체로 변환
- `ResponseEntity`: 응답 상태 코드와 Body 등을 설정
- `.status()`: HTTP 상태 코드 지정
- `HttpStatus.CREATED`: 201 Created
- `.build()`: 설정한 내용으로 응답 객체 생성

<img src="/images/lv3_PlayerController.png" width="800">

3. 이미 등록된 닉네임이면 `DUPLICATE_NICKNAME` 에러 던짐
- nickname에 인덱스가 걸려있지 않으면 Table Scan 발생
- `Table Scan` : 인덱스를 사용하지 않고 테이블의 행을 직접 순차적으로 확인하면서 조건에 맞는 데이터를 찾는 방식이다.

<img src="/images/lv3_createPlayer.png" width="800">

---
### Lv 4. 월드 생성
1. `createWorld()`에서 월드 제한 검사

<img src="/images/lv4_createWorld.png" width="800">

2. 응답 헤더 설정
- `Header`: 응답에 대한 부가 정보(메타데이터)를 전달
- `.headers`: HTTP 응답 Header 설정
- `HttpHeaders`: HTTP Header 정보를 담는 객체

<img src="/images/lv4_WorldController.png" width="800">

---
### Lv 5. 채팅 저장과 내역 조회
1. 메시지 저장
- `.findById()`: ID로 Entity 조회
- `.save()`: Entity 저장
- `ChatMessageResponse`: Entity가 아닌 DTO로 응답

<img src="/images/lv5_saveMessage.png" width="800">

2. 오래된 순서로 응답 DTO 목록 반환
- `.reversed()`: List를 역순으로 보는 `Reverse View`를 반환
- `Reverse View`: 원본 List를 역순으로 바라보는 View. <br> 원본 List와 데이터를 공유하며, 별도의 List를 복사하지 않는다.

<img src="/images/lv5_getRecentMessages.png" width="800">

---
### Lv 6. 최근 채팅 조회 API 구현
1. 최근 채팅 조회 요청 메서드
- `@GetMapping`: GET 요청 URL 지정
- `@PathVariable`: URL 경로의 값을 매개변수로 전달
- `@RequestParam`: Query Parameter를 매개변수로 전달
- `@RequestParam.defaultValue`: `@RequestParam`의 값이 없을 때 사용할 기본값

<img src="/images/lv6_WorldChatController.png" width="800">

2. limit 범위 보정
- `Math.min(a, b)`: 두 값 중 작은 값을 반환
- `Math.max(a, b)`: 두 값 중 큰 값을 반환

<img src="/images/lv6_RecentChatQueryService.png" width="800">

---
### Lv 7. WebSocket 연결과 사용자 식별
1. `WebSocket HandshakeInterceptor` 설정
- `HandshakeInterceptor`는 WebSocket 연결을 시작하는 HTTP Handshake 과정에 개입하는 Spring 인터셉터다.<br>
Handshake 과정에서 요청을 검증하거나 WebSocket 세션에 정보를 저장할 때 사용한다.
- `.beforeHandshake()`: WebSocket 연결 전에 실행
- `WebSocketHandler`: Handshake 이후 WebSocket 연결의 통신을 처리하는 Handler
- `attributes`: Handshake 과정에서 WebSocket 세션에 전달할 데이터를 저장하는 Map
- `.put(key, value)`: Map에 key와 value를 저장
- `return true`: 연결 허용 
- `return false`: 연결 거부

<img src="/images/lv7_NicknameHandshakeInterceptor.png" width="800">

---
### Lv 8. HandshakeInterceptor 등록
1. `HandshakeInterceptor` 등록
- `@Configuration`: 해당 클래스를 Spring 설정 클래스로 등록
- `@EnableWebSocket`: Spring WebSocket 기능 활성화
- `@EnableScheduling`: `@Scheduled`를 이용한 스케줄링 기능 활성화
- `@Scheduled`: 특정 메서드를 정해진 시간이나 주기마다 자동으로 실행하도록 설정하는 Spring 어노테이션이다.
- `WebSocketConfigurer`: WebSocket 관련 설정을 정의하는 인터페이스
- `.registerWebSocketHandlers()`: WebSocket Handler와 연결 경로 등을 등록
- `WebSocketHandlerRegistry`: WebSocket Handler 등록을 관리하는 객체
- `.addHandler()`: WebSocket Handler와 접속 URL을 등록
- `.addInterceptors()`: Handshake 과정에 사용할 Interceptor 등록
- `.setAllowedOriginPatterns()`: WebSocket 연결을 허용할 Origin 패턴 지정
- `Origin`: 요청을 보낸 출처(출처의 프로토콜 + 도메인 + 포트)를 나타내는 HTTP Header다.

<img src="/images/lv8_WebSocketConfig.png" width="800">

---
### Lv 9. 월드별 WebSocket 세션 관리
1. 세션 등록
- `ConcurrentHashMap`: 여러 Thread의 동시 접근을 안전하게 처리하는 Map
- `Entry`: WebSocket 세션 등 접속 정보를 담는 객체
- `.key()`: 닉네임을 Map의 Key로 사용할 형태로 변환
- `AtomicReference`: Lambda 내부에서 외부 지역 변수의 값을 변경하기 위한 객체
- `.compute()`: Key의 값을 조회하고, Lambda 결과로 값을 생성하거나 갱신
- `.putIfAbsent()`: Key가 없을 때만 값을 추가하고, 기존 값은 유지<br>
반환이 null이면 새로 등록, null 아니면 이미 존재함

<img src="/images/lv9_register.png" width="800">

---
### Lv 10. Redis 접속 상태 관리
1. Redis 접속 상태 등록 및 해제
- `RedisTemplate`: Spring에서 Redis와 통신하기 위한 Template 클래스 <br>
Java 메서드를 통해 Redis의 데이터를 조작 <br>
`K`, `V` 타입은 제네릭으로 지정하며, 실제 Redis 저장 시 Serializer를 통해 byte 데이터로 변환 <br>
Redis가 실제로 저장하는 데이터는 byte 데이터
- `StringRedisTemplate`: Key와 Value를 문자열로 다루도록 구성된 `RedisTemplate`
- `.opsForZSet()`: Redis Sorted Set(ZSet) 연산 객체
- `ZSet`: `member`와 `score`를 함께 저장하는 Redis 자료구조
- `TTL`: 개별 접속자의 만료 시간을 계산할 때 사용
- `KEY_TTL`: Redis Key 자체에 설정하는 180초 TTL
- `.removeRangeByScore()`: 특정 score 범위에 해당하는 member를 삭제
- `.zCard()`: ZSet의 member 개수를 반환

<img src="/images/lv10_PresenceService.png" width="800">

---
### Lv 11. 메시지 라우팅과 Ping/Pong
1. 적절한 type에 해당하는 Handler를 호출하는 router
- `ServerHttpRequest`: Spring이 추상화한 HTTP 요청 객체. <br>
WebSocket 핸드셰이크 과정에서 클라이언트의 요청 정보(URI, Header 등)를 다룬다.
- `WebSocketSession`: WebSocket 연결이 수립된 후 클라이언트와의 연결을 나타내고 관리하는 객체. <br>
Spring이 생성하여 Handler에 전달한다.
- `TextMessage`: WebSocket을 통해 전달된 텍스트 메시지를 Spring이 표현하는 객체.<br>
메시지의 문자열 내용은 `.getPayload()`로 가져온다.
- `JsonNode`: 문자열로 전달된 JSON을 Jackson이 파싱하여 만든 객체.<br>
트리 구조(Tree)로 표현

<img src="/images/lv11_route.png" width="800">

2. `type`이 `ping`일 때 Handler 처리

<img src="/images/lv11_PingWsHandler.png" width="800">

---
### Lv 12. 플레이어 이동 요청 처리
1. `type`이 `move`일 때 Handler 처리

<img src="/images/lv12_MoveWsHandler.png" width="800">

---
### Lv 13. 채팅 요청 처리와 응답 구성
1. 채팅 응답 DTO

<img src="/images/lv13_ChatResponse.png" width="800">

2. 메시지의 content 필드 읽고 저장 및 응답 생성

<img src="/images/lv13_ChatWsHandler.png" width="800">

---
### Lv 14. 같은 월드의 참여자에게 채팅 전송
1. Redis의 Pub/Sub을 사용하지 않는 경우 broadcaster으로 메시지 전파

<img src="/images/lv14_LocalChatSender.png" width="800">

---
### Lv 15. 접속자 목록 조회
1`type`이 `onlineUsers`일 때 Handler 처리

<img src="/images/lv15_OnlineUsersWsHandler.png" width="800">

2`type`이 `onlineUsers`일 때 응답 DTO

<img src="/images/lv15_OnlineUsersResponse.png" width="800">
