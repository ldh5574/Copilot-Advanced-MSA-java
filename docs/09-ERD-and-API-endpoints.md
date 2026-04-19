# ERD 및 API 엔드포인트 설명서

## 1. Entity Relationship Diagram (ERD)

### 1.1 전체 마이크로서비스 아키텍처

```
┌─────────────────┐          ┌──────────────────┐          ┌────────────────┐
│  User Service   │          │ Product Service  │          │ Order Service  │
│                 │          │                  │          │                │
│   - User 엔티티 │          │ - Product 엔티티 │          │  - Order 엔티티│
└────────┬────────┘          └────────┬─────────┘          └────────┬───────┘
         │                           │                               │
         │ user-events 토픽          │ stock-events 토픽             │ order-events 토픽
         │ (UserCreatedEvent)        │ (StockUpdatedEvent)           │ (OrderCreatedEvent)
         │                           │                               │
         └───────────────────┬───────┴───────────────────┬──────────┘
                             │                           │
                        ┌────▼────────────────────────────▼───┐
                        │                                      │
                        │          Kafka Message Broker        │
                        │                                      │
                        └──────────────────────────────────────┘
```

---

## 2. 데이터베이스 스키마

### 2.1 User Service - users 테이블

```sql
CREATE TABLE users (
  id BIGINT PRIMARY KEY AUTO_INCREMENT,
  username VARCHAR(255) NOT NULL UNIQUE,
  email VARCHAR(255) NOT NULL UNIQUE,
  password VARCHAR(255) NOT NULL,
  full_name VARCHAR(255),
  phone_number VARCHAR(20),
  created_at DATETIME NOT NULL,
  updated_at DATETIME,
  INDEX idx_username (username),
  INDEX idx_email (email)
);
```

| 컬럼명 | 타입 | 제약조건 | 설명 |
|---------|------|---------|------|
| id | BIGINT | PK, AUTO_INCREMENT | 사용자 고유 ID |
| username | VARCHAR(255) | NOT NULL, UNIQUE | 사용자명 (로그인 ID) |
| email | VARCHAR(255) | NOT NULL, UNIQUE | 이메일 주소 |
| password | VARCHAR(255) | NOT NULL | 암호화된 비밀번호 |
| full_name | VARCHAR(255) | | 실명 |
| phone_number | VARCHAR(20) | | 전화번호 |
| created_at | DATETIME | NOT NULL | 생성 시각 |
| updated_at | DATETIME | | 수정 시각 |

---

### 2.2 Product Service - products 테이블

```sql
CREATE TABLE products (
  id BIGINT PRIMARY KEY AUTO_INCREMENT,
  name VARCHAR(255) NOT NULL,
  description TEXT,
  price DECIMAL(10, 2) NOT NULL,
  stock INT NOT NULL,
  category VARCHAR(100),
  created_at DATETIME NOT NULL,
  updated_at DATETIME,
  INDEX idx_category (category),
  INDEX idx_price (price)
);
```

| 컬럼명 | 타입 | 제약조건 | 설명 |
|---------|------|---------|------|
| id | BIGINT | PK, AUTO_INCREMENT | 상품 고유 ID |
| name | VARCHAR(255) | NOT NULL | 상품명 |
| description | TEXT | | 상품 설명 |
| price | DECIMAL(10, 2) | NOT NULL | 상품 가격 |
| stock | INT | NOT NULL | 현재 재고 |
| category | VARCHAR(100) | | 상품 카테고리 |
| created_at | DATETIME | NOT NULL | 생성 시각 |
| updated_at | DATETIME | | 수정 시각 |

---

### 2.3 Order Service - orders 테이블

```sql
CREATE TABLE orders (
  id BIGINT PRIMARY KEY AUTO_INCREMENT,
  user_id BIGINT NOT NULL,
  product_id BIGINT NOT NULL,
  quantity INT NOT NULL,
  total_price DECIMAL(15, 2) NOT NULL,
  status VARCHAR(50) NOT NULL DEFAULT 'PENDING',
  created_at DATETIME NOT NULL,
  updated_at DATETIME,
  INDEX idx_user_id (user_id),
  INDEX idx_product_id (product_id),
  INDEX idx_status (status),
  FOREIGN KEY (user_id) REFERENCES users(id),
  FOREIGN KEY (product_id) REFERENCES products(id)
);
```

| 컬럼명 | 타입 | 제약조건 | 설명 |
|---------|------|---------|------|
| id | BIGINT | PK, AUTO_INCREMENT | 주문 고유 ID |
| user_id | BIGINT | NOT NULL, FK | 사용자 ID (User Service 참조) |
| product_id | BIGINT | NOT NULL, FK | 상품 ID (Product Service 참조) |
| quantity | INT | NOT NULL | 주문 수량 |
| total_price | DECIMAL(15, 2) | NOT NULL | 주문 총액 |
| status | VARCHAR(50) | NOT NULL, DEFAULT: PENDING | 주문 상태 |
| created_at | DATETIME | NOT NULL | 주문 생성 시각 |
| updated_at | DATETIME | | 주문 수정 시각 |

#### OrderStatus Enum

```java
PENDING,      // 주문 대기 중
CONFIRMED,    // 주문 확정
PROCESSING,   // 처리 중
SHIPPED,      // 배송 중
DELIVERED,    // 배송 완료
CANCELLED     // 취소됨
```

---

## 3. API 엔드포인트

### 3.1 User Service API

#### 3.1.1 사용자 등록
- **Method**: POST
- **URL**: `/api/users`
- **Request Body**:
  ```json
  {
    "username": "john_doe",
    "email": "john@example.com",
    "password": "password123",
    "fullName": "John Doe",
    "phoneNumber": "010-1234-5678"
  }
  ```
- **Response** (201 Created):
  ```json
  {
    "success": true,
    "message": "사용자가 생성되었습니다",
    "data": {
      "id": 1,
      "username": "john_doe",
      "email": "john@example.com",
      "fullName": "John Doe",
      "phoneNumber": "010-1234-5678",
      "createdAt": "2026-03-11T10:30:00"
    }
  }
  ```
- **Kafka Event**: `user-events` 토픽으로 `UserCreatedEvent` 발행

---

#### 3.1.2 사용자 조회 (ID로)
- **Method**: GET
- **URL**: `/api/users/{id}`
- **Path Parameter**: `id` (Long) - 사용자 ID
- **Response** (200 OK):
  ```json
  {
    "success": true,
    "message": "",
    "data": {
      "id": 1,
      "username": "john_doe",
      "email": "john@example.com",
      "fullName": "John Doe",
      "phoneNumber": "010-1234-5678",
      "createdAt": "2026-03-11T10:30:00",
      "updatedAt": "2026-03-11T10:35:00"
    }
  }
  ```

---

#### 3.1.3 전체 사용자 조회
- **Method**: GET
- **URL**: `/api/users`
- **Response** (200 OK):
  ```json
  {
    "success": true,
    "message": "",
    "data": [
      {
        "id": 1,
        "username": "john_doe",
        "email": "john@example.com",
        "fullName": "John Doe",
        "phoneNumber": "010-1234-5678",
        "createdAt": "2026-03-11T10:30:00"
      },
      {
        "id": 2,
        "username": "jane_smith",
        "email": "jane@example.com",
        "fullName": "Jane Smith",
        "phoneNumber": "010-9876-5432",
        "createdAt": "2026-03-11T11:00:00"
      }
    ]
  }
  ```

---

#### 3.1.4 사용자 삭제
- **Method**: DELETE
- **URL**: `/api/users/{id}`
- **Path Parameter**: `id` (Long) - 사용자 ID
- **Response** (200 OK):
  ```json
  {
    "success": true,
    "message": "사용자가 삭제되었습니다",
    "data": null
  }
  ```

---

### 3.2 Product Service API

#### 3.2.1 상품 등록
- **Method**: POST
- **URL**: `/api/products`
- **Request Body**:
  ```json
  {
    "name": "MacBook Pro 16",
    "description": "High-performance laptop",
    "price": 2499.99,
    "stock": 50,
    "category": "Electronics"
  }
  ```
- **Response** (201 Created):
  ```json
  {
    "success": true,
    "message": "상품이 생성되었습니다",
    "data": {
      "id": 1,
      "name": "MacBook Pro 16",
      "description": "High-performance laptop",
      "price": 2499.99,
      "stock": 50,
      "category": "Electronics",
      "createdAt": "2026-03-11T10:30:00"
    }
  }
  ```

---

#### 3.2.2 상품 조회 (ID로)
- **Method**: GET
- **URL**: `/api/products/{id}`
- **Path Parameter**: `id` (Long) - 상품 ID
- **Response** (200 OK):
  ```json
  {
    "success": true,
    "message": "",
    "data": {
      "id": 1,
      "name": "MacBook Pro 16",
      "description": "High-performance laptop",
      "price": 2499.99,
      "stock": 50,
      "category": "Electronics",
      "createdAt": "2026-03-11T10:30:00",
      "updatedAt": "2026-03-11T10:35:00"
    }
  }
  ```

---

#### 3.2.3 전체 상품 조회
- **Method**: GET
- **URL**: `/api/products`
- **Response** (200 OK):
  ```json
  {
    "success": true,
    "message": "",
    "data": [
      {
        "id": 1,
        "name": "MacBook Pro 16",
        "price": 2499.99,
        "stock": 50,
        "category": "Electronics",
        "createdAt": "2026-03-11T10:30:00"
      },
      {
        "id": 2,
        "name": "iPad Air",
        "price": 599.99,
        "stock": 100,
        "category": "Electronics",
        "createdAt": "2026-03-11T10:40:00"
      }
    ]
  }
  ```

---

#### 3.2.4 카테고리별 상품 조회
- **Method**: GET
- **URL**: `/api/products/category/{category}`
- **Path Parameter**: `category` (String) - 상품 카테고리
- **Response** (200 OK):
  ```json
  {
    "success": true,
    "message": "",
    "data": [
      {
        "id": 1,
        "name": "MacBook Pro 16",
        "price": 2499.99,
        "stock": 50,
        "category": "Electronics"
      }
    ]
  }
  ```

---

#### 3.2.5 재고 증가
- **Method**: PATCH
- **URL**: `/api/products/{id}/stock/increase`
- **Path Parameter**: `id` (Long) - 상품 ID
- **Query Parameter**: `quantity` (Integer) - 증가할 수량
- **Response** (200 OK):
  ```json
  {
    "success": true,
    "message": "재고가 증가되었습니다",
    "data": null
  }
  ```
- **Kafka Event**: `stock-events` 토픽으로 `StockUpdatedEvent` 발행

---

#### 3.2.6 재고 감소
- **Method**: PATCH
- **URL**: `/api/products/{id}/stock/decrease`
- **Path Parameter**: `id` (Long) - 상품 ID
- **Query Parameter**: `quantity` (Integer) - 감소할 수량
- **Query Parameter**: `reason` (String) - 감소 사유 (주문, 손상, 기타)
- **Response** (200 OK):
  ```json
  {
    "success": true,
    "message": "재고가 감소되었습니다",
    "data": null
  }
  ```
- **Kafka Event**: `stock-events` 토픽으로 `StockUpdatedEvent` 발행

---

### 3.3 Order Service API

#### 3.3.1 주문 생성
- **Method**: POST
- **URL**: `/api/orders`
- **Request Body**:
  ```json
  {
    "userId": 1,
    "productId": 1,
    "quantity": 2,
    "unitPrice": 2499.99
  }
  ```
- **Response** (201 Created):
  ```json
  {
    "success": true,
    "message": "주문이 생성되었습니다",
    "data": {
      "id": 1,
      "userId": 1,
      "productId": 1,
      "quantity": 2,
      "totalPrice": 4999.98,
      "status": "PENDING",
      "createdAt": "2026-03-11T10:30:00"
    }
  }
  ```
- **Kafka Event**: `order-events` 토픽으로 `OrderCreatedEvent` 발행

---

#### 3.3.2 주문 조회 (ID로)
- **Method**: GET
- **URL**: `/api/orders/{id}`
- **Path Parameter**: `id` (Long) - 주문 ID
- **Response** (200 OK):
  ```json
  {
    "success": true,
    "message": "",
    "data": {
      "id": 1,
      "userId": 1,
      "productId": 1,
      "quantity": 2,
      "totalPrice": 4999.98,
      "status": "CONFIRMED",
      "createdAt": "2026-03-11T10:30:00",
      "updatedAt": "2026-03-11T10:35:00"
    }
  }
  ```

---

#### 3.3.3 전체 주문 조회
- **Method**: GET
- **URL**: `/api/orders`
- **Response** (200 OK):
  ```json
  {
    "success": true,
    "message": "",
    "data": [
      {
        "id": 1,
        "userId": 1,
        "productId": 1,
        "quantity": 2,
        "totalPrice": 4999.98,
        "status": "CONFIRMED",
        "createdAt": "2026-03-11T10:30:00"
      },
      {
        "id": 2,
        "userId": 2,
        "productId": 2,
        "quantity": 1,
        "totalPrice": 599.99,
        "status": "PENDING",
        "createdAt": "2026-03-11T10:45:00"
      }
    ]
  }
  ```

---

#### 3.3.4 사용자별 주문 조회
- **Method**: GET
- **URL**: `/api/orders/user/{userId}`
- **Path Parameter**: `userId` (Long) - 사용자 ID
- **Response** (200 OK):
  ```json
  {
    "success": true,
    "message": "",
    "data": [
      {
        "id": 1,
        "userId": 1,
        "productId": 1,
        "quantity": 2,
        "totalPrice": 4999.98,
        "status": "CONFIRMED",
        "createdAt": "2026-03-11T10:30:00"
      }
    ]
  }
  ```

---

#### 3.3.5 주문 상태 변경
- **Method**: PATCH
- **URL**: `/api/orders/{id}/status`
- **Path Parameter**: `id` (Long) - 주문 ID
- **Query Parameter**: `status` (OrderStatus) - 변경할 상태
  - PENDING, CONFIRMED, PROCESSING, SHIPPED, DELIVERED, CANCELLED
- **Response** (200 OK):
  ```json
  {
    "success": true,
    "message": "주문 상태가 변경되었습니다",
    "data": {
      "id": 1,
      "userId": 1,
      "productId": 1,
      "quantity": 2,
      "totalPrice": 4999.98,
      "status": "SHIPPED",
      "createdAt": "2026-03-11T10:30:00",
      "updatedAt": "2026-03-11T14:00:00"
    }
  }
  ```

---

#### 3.3.6 주문 취소
- **Method**: POST
- **URL**: `/api/orders/{id}/cancel`
- **Path Parameter**: `id` (Long) - 주문 ID
- **Response** (200 OK):
  ```json
  {
    "success": true,
    "message": "주문이 취소되었습니다",
    "data": {
      "id": 1,
      "userId": 1,
      "productId": 1,
      "quantity": 2,
      "totalPrice": 4999.98,
      "status": "CANCELLED",
      "createdAt": "2026-03-11T10:30:00",
      "updatedAt": "2026-03-11T15:00:00"
    }
  }
  ```
- **Kafka Event**: 취소 이벤트를 `order-events` 토픽으로 발행하여 Product Service에서 재고 복구

---

## 4. Kafka 이벤트 통신 구조

### 4.1 Kafka 토픽별 이벤트

#### 4.1.1 user-events 토픽

**Producer**: User Service  
**Consumer**: Order Service

**Event**: UserCreatedEvent
```json
{
  "userId": 1,
  "username": "john_doe",
  "email": "john@example.com",
  "createdAt": "2026-03-11T10:30:00"
}
```

**용도**: Order Service에서 사용자 정보를 캐싱하거나 로컬 저장

---

#### 4.1.2 stock-events 토픽

**Producer**: Product Service  
**Consumer**: Order Service

**Event**: StockUpdatedEvent
```json
{
  "productId": 1,
  "quantity": 10,
  "operation": "DECREASE",
  "reason": "ORDER",
  "updatedAt": "2026-03-11T10:32:00"
}
```

**용도**: Order Service에서 재고 변경을 감지하여 주문 상태 업데이트

---

#### 4.1.3 order-events 토픽

**Producer**: Order Service  
**Consumer**: Product Service

**Event**: OrderCreatedEvent
```json
{
  "orderId": 1,
  "userId": 1,
  "productId": 1,
  "quantity": 2,
  "createdAt": "2026-03-11T10:30:00"
}
```

**용도**: Product Service에서 주문에 따른 재고 차감 처리

---

### 4.2 이벤트 흐름 다이어그램

```
┌──────────────────────────────────────────────────────────────────────┐
│                      주문 생성 프로세스                              │
├──────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  1. Client                                                           │
│     │                                                                │
│     └──> POST /api/orders                                            │
│          │                                                            │
│          ▼                                                            │
│     Order Service                                                    │
│     - Order 엔티티 생성 (status: PENDING)                            │
│     - OrderCreatedEvent 발행                                         │
│          │                                                            │
│          └──> Kafka (order-events 토픽)                              │
│               │                                                       │
│               ▼                                                       │
│          Product Service                                             │
│          - 재고 확인 및 차감                                          │
│          - StockUpdatedEvent 발행                                    │
│               │                                                       │
│               └──> Kafka (stock-events 토픽)                          │
│                    │                                                  │
│                    ▼                                                  │
│               Order Service (Consumer)                               │
│               - StockUpdatedEvent 수신                               │
│               - Order status: PENDING → CONFIRMED로 변경             │
│                                                                      │
└──────────────────────────────────────────────────────────────────────┘
```

---

## 5. API 요청/응답 공통 패턴

### 5.1 성공 응답

```json
{
  "success": true,
  "message": "작업 설명",
  "data": {
    // 실제 데이터
  }
}
```

**HTTP Status**: 200 OK (조회, 수정), 201 Created (생성)

---

### 5.2 에러 응답

```json
{
  "success": false,
  "message": "오류 설명",
  "data": null
}
```

**HTTP Status 예시**:
- 400 Bad Request: 요청 데이터 검증 오류
- 404 Not Found: 리소스를 찾을 수 없음
- 500 Internal Server Error: 서버 오류

---

## 6. 요구사항 정리표

| 서비스 | 요구사항 | API 엔드포인트 | Kafka 이벤트 |
|--------|---------|----------------|-------------|
| **User Service** | 사용자 등록 | POST `/api/users` | `user-events` → UserCreatedEvent |
| | 사용자 조회 | GET `/api/users/{id}` | - |
| | 전체 사용자 조회 | GET `/api/users` | - |
| | 사용자 삭제 | DELETE `/api/users/{id}` | - |
| **Product Service** | 상품 등록 | POST `/api/products` | - |
| | 상품 조회 | GET `/api/products/{id}` | - |
| | 전체 상품 조회 | GET `/api/products` | - |
| | 카테고리별 조회 | GET `/api/products/category/{category}` | - |
| | 재고 관리 (증가) | PATCH `/api/products/{id}/stock/increase` | `stock-events` → StockUpdatedEvent |
| | 재고 관리 (감소) | PATCH `/api/products/{id}/stock/decrease` | `stock-events` → StockUpdatedEvent |
| **Order Service** | 주문 생성 | POST `/api/orders` | `order-events` → OrderCreatedEvent |
| | 주문 조회 | GET `/api/orders/{id}` | - |
| | 전체 주문 조회 | GET `/api/orders` | - |
| | 사용자별 주문 조회 | GET `/api/orders/user/{userId}` | - |
| | 주문 상태 변경 | PATCH `/api/orders/{id}/status` | - |
| | 주문 취소 | POST `/api/orders/{id}/cancel` | `order-events` → OrderCancelledEvent |

---

## 7. 주요 특징 및 주의사항

### 7.1 데이터 일관성
- 주문 생성 시 Order Service에서 기록하고, Product Service에서 재고를 감소시키기 위해 Kafka 이벤트 기반 비동기 처리
- 이벤트 처리 실패 시 retry 메커니즘 필요

### 7.2 서비스 간 데이터 참조
- Order Service의 `userId`, `productId`는 각각 User Service, Product Service의 id를 참조
- Foreign Key 제약조건은 데이터베이스에 설정하나, 마이크로서비스 특성상 실제 조회는 각 서비스의 API 호출 필요

### 7.3 보안 고려사항
- User Service의 password는 실제 운영 환경에서 암호화 필요 (현재: 주석 표시)
- API 인증/인가 메커니즘 필요 (JWT, OAuth 등)
- Kafka 메시지는 민감 정보 제외

---

## 8. 확장 시 고려사항

- **Payment Service**: 결제 처리를 위한 별도 서비스 추가 가능
- **Notification Service**: 주문/재고 변경 알림 처리
- **Analytics Service**: 주문, 판매 현황 분석
- **Distributed Tracing**: 서비스 간 요청 추적을 위해 Spring Cloud Sleuth, Jaeger 등 도입
