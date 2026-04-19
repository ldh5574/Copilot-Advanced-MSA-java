# 10. 서비스 구현 계획서

## 🎯 목표

Spring Boot 3.2 기반의 마이크로서비스 아키텍처 프로젝트를 완전히 새로 구현합니다. 4개의 서비스(API Gateway, User Service, Product Service, Order Service)를 단계별로 구축하며, Kafka를 통한 이벤트 기반 비동기 통신과 H2 메모리 DB를 사용합니다.

---

## 📋 구현 단계별 계획

### **Phase 1: Common 모듈 구축 (의존성 없음)**

**목적**: 모든 서비스에서 공통으로 사용할 DTO 및 이벤트 클래스 정의

#### 1.1 빌드 설정
- `common/build.gradle.kts` 생성 및 의존성 설정
  - Spring Data JPA, Lombok, Jackson 등

#### 1.2 공통 응답 형식
- **경로**: `common/src/main/java/com/example/common/dto/ApiResponse.java`
- **구조**: 모든 REST API 응답을 통일된 형식으로 제공
  ```json
  {
    "success": true,
    "message": "성공 메시지",
    "data": { ... }
  }
  ```

#### 1.3 Kafka 이벤트 클래스
- `common/src/main/java/com/example/common/event/UserCreatedEvent.java`
  - userId, username, email, createdAt
  
- `common/src/main/java/com/example/common/event/OrderCreatedEvent.java`
  - orderId, userId, productId, quantity, createdAt
  
- `common/src/main/java/com/example/common/event/StockUpdatedEvent.java`
  - productId, productName, previousStock, currentStock, changeAmount, reason, updatedAt

#### 1.4 공통 엔티티 매핑
- `common` 모듈에 필요한 공통 인터페이스나 기본 클래스 정의

---

### **Phase 2: User Service 구현 (포트: 8081)**

**목적**: 사용자 관리 및 사용자 생성 이벤트 발행

#### 2.1 프로젝트 초기 설정
- `user-service/build.gradle.kts` 작성
  - Spring Boot 3.2, Spring Data JPA, Apache Kafka, H2 Database, Lombok
- `user-service/src/main/java/com/example/userservice/UserServiceApplication.java`
  - Main 클래스 및 @SpringBootApplication 어노테이션
- `user-service/src/main/resources/application.yml`
  - 포트: 8081
  - H2 DB 설정: `jdbc:h2:mem:userdb` (메모리 DB)
  - Kafka Bootstrap Servers 설정

#### 2.2 데이터 모델
- **엔티티**: `user-service/src/main/java/com/example/userservice/entity/User.java`
  - id (Long, PK, AUTO_INCREMENT)
  - username (String, NOT NULL, UNIQUE)
  - email (String, NOT NULL, UNIQUE)
  - password (String, NOT NULL)
  - fullName (String)
  - phoneNumber (String)
  - createdAt (LocalDateTime, NOT NULL)
  - updatedAt (LocalDateTime)

- **Repository**: `user-service/src/main/java/com/example/userservice/repository/UserRepository.java`
  - extends JpaRepository<User, Long>
  - 메서드: findByUsername, findByEmail, existsByUsername, existsByEmail

#### 2.3 API 레이어
- **DTO**:
  - `user-service/src/main/java/com/example/userservice/dto/CreateUserRequest.java`
  - `user-service/src/main/java/com/example/userservice/dto/UserResponse.java`

- **Controller**: `user-service/src/main/java/com/example/userservice/controller/UserController.java`
  - `POST /api/users` - 사용자 생성
  - `GET /api/users/{id}` - 사용자 조회
  - `GET /api/users` - 전체 사용자 조회
  - `DELETE /api/users/{id}` - 사용자 삭제

#### 2.4 비즈니스 로직
- **Service**: `user-service/src/main/java/com/example/userservice/service/UserService.java`
  - createUser(): 사용자 생성 (중복 검사)
  - getUserById(): 사용자 조회
  - getAllUsers(): 전체 사용자 조회
  - deleteUser(): 사용자 삭제
  - 트랜잭션 관리 (@Transactional)

#### 2.5 Kafka Integration
- **Producer**: `user-service/src/main/java/com/example/userservice/kafka/UserEventProducer.java`
  - 사용자 생성 시 `UserCreatedEvent` 발행
  - 토픽: `user-events`

#### 2.6 구성 설정
- **Kafka Config**: `user-service/src/main/java/com/example/userservice/config/KafkaConfig.java`
  - KafkaTemplate 빈 설정
  - Topic 설정

---

### **Phase 3: Product Service 구현 (포트: 8082)**

**목적**: 상품 관리 및 재고 관리

#### 3.1 프로젝트 초기 설정
- `product-service/build.gradle.kts` 작성
- `product-service/src/main/java/com/example/productservice/ProductServiceApplication.java`
- `product-service/src/main/resources/application.yml`
  - 포트: 8082
  - H2 DB 설정: `jdbc:h2:mem:productdb`

#### 3.2 데이터 모델
- **엔티티**: `product-service/src/main/java/com/example/productservice/entity/Product.java`
  - id (Long, PK, AUTO_INCREMENT)
  - name (String, NOT NULL)
  - description (String)
  - price (BigDecimal, NOT NULL)
  - stock (Integer, NOT NULL)
  - category (String)
  - createdAt (LocalDateTime, NOT NULL)
  - updatedAt (LocalDateTime)

- **Repository**: `product-service/src/main/java/com/example/productservice/repository/ProductRepository.java`
  - 메서드: findByCategory, findByNameLike 등

#### 3.3 API 레이어
- **DTO**:
  - `product-service/src/main/java/com/example/productservice/dto/CreateProductRequest.java`
  - `product-service/src/main/java/com/example/productservice/dto/ProductResponse.java`

- **Controller**: `product-service/src/main/java/com/example/productservice/controller/ProductController.java`
  - `POST /api/products` - 상품 생성
  - `GET /api/products/{id}` - 상품 조회
  - `GET /api/products` - 전체 상품 조회
  - `GET /api/products/category/{category}` - 카테고리별 상품 조회

#### 3.4 비즈니스 로직
- **Service**: `product-service/src/main/java/com/example/productservice/service/ProductService.java`
  - createProduct(): 상품 생성
  - getProductById(): 상품 조회
  - getAllProducts(): 전체 상품 조회
  - getProductsByCategory(): 카테고리별 상품 조회
  - decreaseStock(): 재고 감소 (Order Service 이벤트 처리)
  - increaseStock(): 재고 증가

#### 3.5 Kafka Integration
- **Consumer**: `product-service/src/main/java/com/example/productservice/kafka/OrderEventConsumer.java`
  - `OrderCreatedEvent` 구독 (토픽: `order-events`)
  - 주문 생성 시 재고 차감 로직 처리

- **Producer**: `product-service/src/main/java/com/example/productservice/kafka/ProductEventProducer.java`
  - 재고 변경 시 `StockUpdatedEvent` 발행
  - 토픽: `stock-events`

#### 3.6 구성 설정
- **Kafka Config**: `product-service/src/main/java/com/example/productservice/config/KafkaConfig.java`
  - Consumer 그룹 설정: `product-service`

---

### **Phase 4: Order Service 구현 (포트: 8083)**

**목적**: 주문 관리 및 주문 상태 추적

#### 4.1 프로젝트 초기 설정
- `order-service/build.gradle.kts` 작성
- `order-service/src/main/java/com/example/orderservice/OrderServiceApplication.java`
- `order-service/src/main/resources/application.yml`
  - 포트: 8083
  - H2 DB 설정: `jdbc:h2:mem:orderdb`

#### 4.2 데이터 모델
- **Enum**: `order-service/src/main/java/com/example/orderservice/entity/OrderStatus.java`
  - PENDING (주문 대기)
  - CONFIRMED (주문 확정)
  - PROCESSING (처리 중)
  - SHIPPED (배송 중)
  - DELIVERED (배송 완료)
  - CANCELLED (취소됨)

- **엔티티**: `order-service/src/main/java/com/example/orderservice/entity/Order.java`
  - id (Long, PK, AUTO_INCREMENT)
  - userId (Long, NOT NULL)
  - productId (Long, NOT NULL)
  - quantity (Integer, NOT NULL)
  - totalPrice (BigDecimal, NOT NULL)
  - status (OrderStatus, NOT NULL, DEFAULT: PENDING)
  - createdAt (LocalDateTime, NOT NULL)
  - updatedAt (LocalDateTime)

- **Repository**: `order-service/src/main/java/com/example/orderservice/repository/OrderRepository.java`
  - 메서드: findByUserId, findByStatus 등

#### 4.3 API 레이어
- **DTO**:
  - `order-service/src/main/java/com/example/orderservice/dto/CreateOrderRequest.java`
  - `order-service/src/main/java/com/example/orderservice/dto/OrderResponse.java`

- **Controller**: `order-service/src/main/java/com/example/orderservice/controller/OrderController.java`
  - `POST /api/orders` - 주문 생성
  - `GET /api/orders/{id}` - 주문 조회
  - `GET /api/orders` - 전체 주문 조회
  - `GET /api/orders/user/{userId}` - 사용자별 주문 조회
  - `PATCH /api/orders/{id}/status` - 주문 상태 업데이트
  - `DELETE /api/orders/{id}` - 주문 취소

#### 4.4 비즈니스 로직
- **Service**: `order-service/src/main/java/com/example/orderservice/service/OrderService.java`
  - createOrder(): 주문 생성
  - getOrderById(): 주문 조회
  - getAllOrders(): 전체 주문 조회
  - getOrdersByUserId(): 사용자별 주문 조회
  - updateOrderStatus(): 주문 상태 업데이트
  - cancelOrder(): 주문 취소

#### 4.5 Kafka Integration
- **Producer**: `order-service/src/main/java/com/example/orderservice/kafka/OrderEventProducer.java`
  - 주문 생성 시 `OrderCreatedEvent` 발행
  - 토픽: `order-events`

- **Consumer**: `order-service/src/main/java/com/example/orderservice/kafka/EventConsumer.java`
  - `UserCreatedEvent` 구독 (토픽: `user-events`)
  - `StockUpdatedEvent` 구독 (토픽: `stock-events`)
  - 이벤트 수신 후 필요한 비즈니스 로직 처리

#### 4.6 구성 설정
- **Kafka Config**: `order-service/src/main/java/com/example/orderservice/config/KafkaConfig.java`
  - Consumer 그룹 설정: `order-service`

---

### **Phase 5: API Gateway 구현 (포트: 8080)**

**목적**: 클라이언트 요청을 해당 마이크로서비스로 라우팅

#### 5.1 프로젝트 설정 검증
- `api-gateway/build.gradle.kts` 확인 및 필요시 수정
  - Spring Cloud Gateway, Spring Cloud Config Client

#### 5.2 게이트웨이 설정
- `api-gateway/src/main/java/com/example/gateway/ApiGatewayApplication.java` (기존 파일 검토)
- `api-gateway/src/main/resources/application.yml` 최적화
  - 포트: 8080
  - 라우팅 규칙:
    ```yaml
    spring:
      cloud:
        gateway:
          routes:
            - id: user-service
              uri: http://localhost:8081
              predicates:
                - Path=/api/users/**
              
            - id: product-service
              uri: http://localhost:8082
              predicates:
                - Path=/api/products/**
              
            - id: order-service
              uri: http://localhost:8083
              predicates:
                - Path=/api/orders/**
    ```

#### 5.3 CORS 설정
- 클라이언트 요청 허용 설정
- 필요시 예약 서버 주소 설정

#### 5.4 Docker 설정
- `api-gateway/Dockerfile` (기존 파일 검토)

---

### **Phase 6: 테스트 및 통합 검증**

**목적**: 모든 서비스의 기능 및 통합 검증

#### 6.1 Unit Tests (단위 테스트)
- 각 서비스의 **Service** 계층 테스트
  - `service/UserServiceTest.java`
  - `service/ProductServiceTest.java`
  - `service/OrderServiceTest.java`
  
- 메서드명 규칙: `should_기대결과_when_조건`

- 테스트 구조 (AAA Pattern):
  ```java
  @Test
  void should_create_user_when_valid_request() {
    // Arrange - 테스트 데이터 준비
    // Act - 메서드 실행
    // Assert - 결과 검증
  }
  ```

#### 6.2 Repository Tests (저장소 테스트)
- `@DataJpaTest` 사용
- 복잡한 쿼리 및 데이터 접근 로직 테스트

#### 6.3 Controller Tests (API 엔드포인트 테스트)
- `@WebMvcTest` 또는 `@SpringBootTest` 사용
- 각 엔드포인트 동작 검증
- HTTP 상태 코드 및 응답 본문 검증

#### 6.4 Integration Tests (통합 테스트)
- `@SpringBootTest` + `@AutoConfigureMockMvc` 사용
- 실제 데이터베이스와의 상호작용 테스트

#### 6.5 Kafka 통신 검증
- Event Producer/Consumer 통합 테스트
- 이벤트 발행 및 수신 검증
- 메시지 포맷 및 처리 로직 검증

#### 6.6 End-to-End (E2E) 테스트 (선택사항)
- 사용자 생성 → 주문 생성 → 재고 차감 전체 흐름 검증
- 게이트웨이를 통한 API 호출 검증

---

## 🗂️ 핵심 파일 목록

### Common 모듈
```
common/
├── build.gradle.kts
└── src/main/java/com/example/common/
    ├── dto/
    │   └── ApiResponse.java
    └── event/
        ├── UserCreatedEvent.java
        ├── OrderCreatedEvent.java
        └── StockUpdatedEvent.java
```

### User Service
```
user-service/
├── build.gradle.kts
├── Dockerfile
└── src/main/java/com/example/userservice/
    ├── UserServiceApplication.java
    ├── config/
    │   └── KafkaConfig.java
    ├── controller/
    │   └── UserController.java
    ├── service/
    │   └── UserService.java
    ├── repository/
    │   └── UserRepository.java
    ├── entity/
    │   └── User.java
    ├── dto/
    │   ├── CreateUserRequest.java
    │   └── UserResponse.java
    ├── kafka/
    │   └── UserEventProducer.java
    └── resources/
        └── application.yml
```

### Product Service
```
product-service/
├── build.gradle.kts
├── Dockerfile
└── src/main/java/com/example/productservice/
    ├── ProductServiceApplication.java
    ├── config/
    │   └── KafkaConfig.java
    ├── controller/
    │   └── ProductController.java
    ├── service/
    │   └── ProductService.java
    ├── repository/
    │   └── ProductRepository.java
    ├── entity/
    │   └── Product.java
    ├── dto/
    │   ├── CreateProductRequest.java
    │   └── ProductResponse.java
    ├── kafka/
    │   ├── OrderEventConsumer.java
    │   └── ProductEventProducer.java
    └── resources/
        └── application.yml
```

### Order Service
```
order-service/
├── build.gradle.kts
├── Dockerfile
└── src/main/java/com/example/orderservice/
    ├── OrderServiceApplication.java
    ├── config/
    │   └── KafkaConfig.java
    ├── controller/
    │   └── OrderController.java
    ├── service/
    │   └── OrderService.java
    ├── repository/
    │   └── OrderRepository.java
    ├── entity/
    │   ├── Order.java
    │   └── OrderStatus.java
    ├── dto/
    │   ├── CreateOrderRequest.java
    │   └── OrderResponse.java
    ├── kafka/
    │   ├── OrderEventProducer.java
    │   └── EventConsumer.java
    └── resources/
        └── application.yml
```

### API Gateway
```
api-gateway/
├── build.gradle.kts
├── Dockerfile
└── src/main/
    ├── java/com/example/gateway/
    │   └── ApiGatewayApplication.java
    └── resources/
        └── application.yml
```

---

## ✅ 검증 체크리스트

### 1️⃣ 개별 서비스 실행 검증
- [ ] Common 모듈 빌드 성공: `./gradlew :common:build`
- [ ] User Service 실행: `./gradlew :user-service:bootRun`
  - [ ] 헬스체크: `curl http://localhost:8081/actuator/health`
  - [ ] API 테스트: `POST /api/users`
  
- [ ] Product Service 실행: `./gradlew :product-service:bootRun`
  - [ ] 헬스체크: `curl http://localhost:8082/actuator/health`
  - [ ] API 테스트: `POST /api/products`
  
- [ ] Order Service 실행: `./gradlew :order-service:bootRun`
  - [ ] 헬스체크: `curl http://localhost:8083/actuator/health`
  - [ ] API 테스트: `POST /api/orders`

### 2️⃣ API Gateway 라우팅 검증
- [ ] Gateway 실행: `./gradlew :api-gateway:bootRun`
- [ ] User Service 라우팅: `curl http://localhost:8080/api/users`
- [ ] Product Service 라우팅: `curl http://localhost:8080/api/products`
- [ ] Order Service 라우팅: `curl http://localhost:8080/api/orders`

### 3️⃣ Kafka 이벤트 통신 검증
- [ ] Kafka 실행 확인: `docker-compose up` 또는 별도 Kafka 서버
- [ ] 사용자 생성 → `UserCreatedEvent` 발행 확인
  - Kafka UI (localhost:8090) 확인
- [ ] 주문 생성 → `OrderCreatedEvent` 발행 확인
- [ ] Product Service 수신 → 재고 차감 확인
- [ ] `StockUpdatedEvent` → Order Service 수신 확인

### 4️⃣ 통합 테스트 실행
- [ ] 모든 테스트 실행: `./gradlew test`
- [ ] 테스트 커버리지 확인: 80% 이상

### 5️⃣ Docker Compose 통합 검증 (선택사항)
- [ ] 모든 서비스를 한 번에 실행: `docker-compose up`
- [ ] 모든 서비스 정상 실행 확인
- [ ] 각 서비스 간 통신 검증

---

## 🎯 핵심 기술 결정 사항

| 항목 | 선택사항 | 근거 |
|------|---------|------|
| 구현 방식 | 처음부터 새로 구현 | 완전한 이해와 학습 목표 달성 |
| 모듈 구성 | 4개 서비스 + API Gateway | 완전한 MSA 시스템 구축 |
| 데이터베이스 | H2 메모리 DB (서비스 별 독립) | 로컬 개발 환경 용이, 서비스 독립성 보장 |
| 이벤트 통신 | Apache Kafka | 비동기 마이크로서비스 통신 표준 |
| 테스트 프레임워크 | JUnit 5 + Mockito | Spring Boot 표준 및 모던 테스트 라이브러리 |
| API 응답 형식 | ApiResponse<T> 통일 | 일관된 클라이언트 응답 처리 |
| 로그 관리 | Slf4j + Logback | Spring Boot 표준 로깅 |

---

## 📌 추가 고려사항

### 실행 순서
1. **Phase 1** → **Phase 2** → **Phase 3** → **Phase 4** → **Phase 5** → **Phase 6**
2. 각 Phase는 순차적으로 진행 필요 (병렬 구현 불가)
3. 각 Phase 완료 후 즉시 테스트 및 빌드 검증

### 빌드 및 배포
- 각 서비스 구현 후: `./gradlew :서비스명:build`
- 전체 프로젝트 빌드: `./gradlew build`
- Docker 이미지 빌드 및 배포: `docker-compose build && docker-compose up`

### Kafka Topic 자동 생성
- Spring Boot 설정에서 자동 생성 활성화
- 필요시 수동으로 토픽 생성:
  ```bash
  kafka-topics.sh --create --topic user-events --bootstrap-server localhost:9092
  kafka-topics.sh --create --topic order-events --bootstrap-server localhost:9092
  kafka-topics.sh --create --topic stock-events --bootstrap-server localhost:9092
  ```

### API 응답 포맷
- 모든 REST API 응답: `ApiResponse<T>` 형식 통일
- 성공 응답:
  ```json
  {
    "success": true,
    "message": "작업 완료",
    "data": { ... }
  }
  ```
- 실패 응답:
  ```json
  {
    "success": false,
    "message": "오류 메시지",
    "data": null
  }
  ```

### 에러 처리
- GlobalExceptionHandler 구현 권장
- 모든 예외를 중앙에서 처리하여 일관성 있는 응답 제공

### 로깅
- `@Slf4j` 어노테이션 사용
- 메서드 시작/종료, 에러 발생 시점에 로그 기록
- 로그 레벨: DEBUG (개발), INFO (운영)

---

## 📚 참고 자료

- [Copilot Instructions 가이드](./.github/copilot-instructions.md)
- [테스트 가이드](./07-testing-guide.md)
- [코드 리뷰 가이드](./06-code-review-guide.md)
- [ERD 및 API 엔드포인트](./09-ERD-and-API-endpoints.md)
- [Spring Boot 공식 문서](https://spring.io/projects/spring-boot)
- [마이크로서비스 패턴](https://microservices.io/)

---

**상태**: 📋 계획 완료, ✅ 구현 단계로 진행 가능

**최종 업데이트**: 2026-03-11
