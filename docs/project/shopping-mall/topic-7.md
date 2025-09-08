---
layout: default
title: "[cart] 같은 상품은 한번만 담기"
parent: shop
nav_order: 7
---

# 장바구니 중복 방지

### 1) 현재 shop_cart 스키마 요약
```sql
CREATE TABLE `shop_cart` (
  `seq_no` int unsigned NOT NULL AUTO_INCREMENT,
  `user_id` varchar(20) NOT NULL,       -- 회원 ID
  `product_id` varchar(30) NOT NULL,    -- 상품 ID
  `quantity` varchar(45) NOT NULL,      -- ❗ 수량이 문자열(varchar)로 저장됨
  `create_date` datetime NOT NULL DEFAULT CURRENT_TIMESTAMP,
  `update_date` datetime NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  PRIMARY KEY (`seq_no`)
) ENGINE=InnoDB;
```


* 기본키는 seq_no (행 고유키)

* 아직 (user_id, product_id) 유니크 제약 없음   
  → 같은 상품이 같은 유저 장바구니에 중복행으로 들어갈 수 있음

* quantity가 문자열 타입   
  → 수량 연산/검증에 불리하고 데이터혼입 위험

### 2) 질문하신 ALTER 의미
  - **A) 유니크 키 추가**  
```sql
ALTER TABLE shop_cart
  ADD UNIQUE KEY IF NOT EXISTS uq_user_product (user_id, product_id);
```


* 의도: 한 유저가 같은 상품을 장바구니에 한 번만 담을 수 있게 강제  
→ 중복 담기 시, “행 추가”가 아니라 “수량 가산/변경” 로직으로 유도

* 효과: 데이터 무결성/중복 방지, 애플리케이션 로직 단순화

⚠️ 주의: MySQL에서는 ALTER TABLE ... ADD UNIQUE KEY IF NOT EXISTS가 버전에 따라 안 될 수 있습니다.   (많은 환경에서 IF NOT EXISTS 미지원)
