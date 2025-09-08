---
layout: default
title: "[db] CTE"
parent: shop
nav_order: 11
---

# CTE(Common Table Expression, 공통 테이블 식)

CTE(Common Table Expression, 공통 테이블 식) 라고 부릅니다.  
MySQL 8.0부터 지원되기 시작했고, Oracle, SQL Server, PostgreSQL 등에서는 오래 전부터 쓰이던 문법이에요.

### 📌 CTE (WITH ... AS ...)란?

- 일시적으로 이름이 붙은 가상 테이블을 만드는 구문

- WITH 이름 AS (SELECT ...) 형태로 작성하고, 그 이름을 뒤에서 일반 테이블처럼 SELECT / JOIN 할 수 있습니다.

- 복잡한 SQL을 쪼개서 가독성을 높이는 데 많이 사용합니다.

### 📊 예시 (단순 버전)
```sql
WITH temp AS (
  SELECT 1 AS num
  UNION ALL
  SELECT 2
)
SELECT num, num*10 AS num_x10
FROM temp;
```

결과:

|num|	num_x10|
|---|---|
|1|	10|
|2|	20|

👉 여기서 temp가 일시적인 테이블처럼 동작합니다.

### 📊 장바구니 예시에서 쓴 CTE
```sql
WITH
cart_items AS (
    SELECT
        c.product_id,
        c.quantity,
        v.effective_price,
        p.delivery_policy_id,
        dp.type,
        dp.base_cost,
        dp.free_over
    FROM shop_cart c
    JOIN v_products_effective v ON v.product_id = c.product_id
    JOIN shop_products p        ON p.product_id = c.product_id
    LEFT JOIN shop_delivery_policy dp ON dp.id = p.delivery_policy_id
    WHERE c.user_id = p_user_id
),
grouped AS (
    SELECT
        delivery_policy_id,
        type,
        base_cost,
        free_over,
        SUM(effective_price * quantity) AS group_subtotal
    FROM cart_items
    GROUP BY delivery_policy_id, type, base_cost, free_over
)
SELECT * FROM grouped;
```

### 👉 흐름 정리:

1. cart_items: 유저 장바구니 + 상품 + 배송정책을 합쳐 임시 테이블 생성

2. grouped: 그걸 다시 GROUP BY 해서 배송정책별 소계 계산

3. 마지막 SELECT: grouped 결과 조회

### ✅ 정리

- WITH ... AS (...) = 임시 테이블 정의

- 복잡한 SELECT를 여러 단계로 나누어 작성할 수 있음

- MySQL 8.0 이상에서는 완전히 지원 → 지금 쓰신 MySQL 8 환경에서 사용 가능