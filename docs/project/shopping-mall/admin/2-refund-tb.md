---
layout: default
title: "[admin] 반품 테이블"
parent: admin
nav_order: 2
---

# 반품관리 테이블 설계


# 📦 쇼핑몰 사이트 – 반품 테이블 설계서

## ✅ 테이블명: `returns`

고객의 반품 요청부터 관리자 승인, 회수 및 환불까지 전체 반품 프로세스를 저장하고 관리하는 테이블입니다.

---

## 📁 필드 정의

| 필드명               | 타입                      | 설명                                                                                |
| ----------------- | ----------------------- | --------------------------------------------------------------------------------- |
| `id`              | BIGINT (PK)             | 반품 고유 ID                                                                          |
| `user_id`         | BIGINT (FK)             | 고객 ID (`users.id`)                                                                |
| `order_id`        | BIGINT (FK)             | 주문 ID (`orders.id`)                                                               |
| `order_item_id`   | BIGINT (FK)             | 주문 상품 ID (`order_items.id`)                                                       |
| `reason`          | TEXT                    | 반품 사유 (고객 입력)                                                                     |
| `description`     | TEXT (nullable)         | 추가 설명 (선택 입력)                                                                     |
| `status`          | ENUM                    | 반품 상태 (`requested`, `approved`, `rejected`, `collected`, `refunded`, `completed`) |
| `request_date`    | DATETIME                | 반품 요청 일시                                                                          |
| `processed_date`  | DATETIME (nullable)     | 관리자 처리 일시                                                                         |
| `return_method`   | VARCHAR(50)             | 회수 방식 (`courier_pickup`, `direct_send`)                                           |
| `tracking_number` | VARCHAR(100) (nullable) | 반품 송장 번호                                                                          |
| `return_address`  | TEXT (nullable)         | 회수지 주소                                                                            |
| `refund_amount`   | DECIMAL(10,2)           | 환불 금액                                                                             |
| `refund_method`   | VARCHAR(50)             | 환불 방식 (`card`, `account`, `point`)                                                |
| `refund_date`     | DATETIME (nullable)     | 환불 처리 일시                                                                          |
| `image_url`       | VARCHAR(255) (nullable) | 첨부 이미지 (예: 불량 사진)                                                                 |
| `admin_memo`      | TEXT (nullable)         | 관리자 내부 메모                                                                         |
| `created_at`      | DATETIME                | 생성 일시                                                                             |
| `updated_at`      | DATETIME                | 수정 일시                                                                             |
|                   |                         |                                                                                   |

---

## 🔄 반품 상태 ENUM (`status`)

| 값           | 설명               |
|--------------|--------------------|
| `requested`   | 고객이 반품 요청함 |
| `approved`    | 관리자가 반품 승인 |
| `rejected`    | 반품 거절됨        |
| `collected`   | 상품 회수 완료됨   |
| `refunded`    | 환불 완료됨        |
| `completed`   | 반품 전체 처리 완료|

---

## 🔗 외래키 관계

```plaintext
returns.user_id        → users.id  
returns.order_id       → orders.id  
returns.order_item_id  → order_items.id
```


## 인덱스 추천
```sql
CREATE INDEX idx_returns_user_id ON returns(user_id);
CREATE INDEX idx_returns_order_id ON returns(order_id);
CREATE INDEX idx_returns_status ON returns(status);
```

## 연관 테이블

| 테이블명               | 설명              |
| ------------------ | --------------- |
| member             | 회원정보            |
| shop_orders        | 주문 기본 정보        |
| shop_order_product | 주문 내 개별 상품 정보   |
| shop_return_logs   | 반품 상태 변경 이력(선택) |
| shop_refunds       | 환불 상세 처리 이력(선택) |

## 확장 고려 사항
* return_logs로 반품 처리 단계별 기록 저장
* return_reasons 테이블로 사유 관리(코드 테이블 형태)
* refunds 테이블과 연계하여 환불 이력 관리
