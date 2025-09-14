---
layout: default
title: "[db] 택배사"
parent: shop
nav_order: 12
---

# 쇼핑몰 택배사 테이블 설계

Here are couriers developmnet tips.

### 1. 택배사 테이블

```php
CREATE TABLE test
```

```sql
CREATE TABLE shop_couriers (
    id INT AUTO_INCREMENT PRIMARY KEY,    
    name VARCHAR(50) NOT NULL COMMENT '택배사 이름',
    code VARCHAR(20) NOT NULL COMMENT '택배사 코드(시스템용)',
    tracking_url VARCHAR(255) NULL COMMENT '배송조회 URL 패턴',
    is_use TINYINT DEFAULT 1 COMMENT '사용 여부 (1=사용,0=미사용)',
    reg_date TIMESTAMP DEFAULT CURRENT_TIMESTAMP
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_general_ci COMMENT='택배사 목록';
```

#### INSERT 예시
```sql
INSERT INTO shop_couriers (name, code, tracking_url) VALUES
('CJ대한통운', 'CJ', 'https://trace.cjlogistics.com/{tracking_no}'),
('롯데택배', 'LOTTE', 'https://www.lotteglogis.com/open/tracking?invno={tracking_no}'),
('한진택배', 'HANJIN', 'https://www.hanjin.co.kr/Delivery_html/inquiry/result_waybill.jsp?wbl_num={tracking_no}');
```

### 📌 조회 동작 흐름

#### 주문 상세 화면 택배사 정보 표기
```sql
SELECT o.id, o.tracking_no, c.name AS courier_name, c.tracking_url
FROM shop_orders o
LEFT JOIN couriers c ON o.courier_id = c.id
WHERE o.id = :order_id;
```

* 고객 마이페이지: CJ대한통운 / 123456789 처럼 표시
* “배송조회” 버튼 클릭 시 → c.tracking_url 내 {tracking_no} 를 o.tracking_no 로 치환해서 링크 생성
