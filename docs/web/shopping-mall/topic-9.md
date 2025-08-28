---
layout: default
title: "[db] COALESCE()"
parent: shop
grand_parent: Web
nav_order: 9
---

# fallback

### 1️⃣ COALESCE() 함수란?

* SQL 표준 함수

* 인자로 받은 값들 중에서 NULL이 아닌 첫 번째 값을 반환합니다.

* 모든 값이 NULL이라면, 마지막 인자를 반환합니다.

예:
```sql
SELECT COALESCE(NULL, NULL, 5, 10); -- 결과: 5
SELECT COALESCE(NULL, 'apple', 'banana'); -- 결과: 'apple'
```

### 2️⃣ COALESCE(p.final_price, p.sale_price, p.buy_price, 0)

shop_products 테이블의 세 가지 가격 필드를 순서대로 체크합니다.

1. p.final_price

    “최종 판매가” (할인까지 적용된 실제 결제 가격)

    있으면 이것이 우선순위 1

2. p.sale_price

    “판매가” (정가 or 기본 판매 가격)

    final_price가 비어 있으면 사용

3. p.buy_price

    “매입가” (원가)

    sale_price까지 없으면 fallback으로 사용

4. 0

    위 값들이 모두 NULL이면 0 반환 (안전장치)

### 3️⃣ 왜 이렇게 설계했을까?

* 상품 가격 정책이 바뀌어도 항상 “표시할 가격”을 뽑아내야 하기 때문입니다.

* 관리자가 최종가만 입력했을 수도 있고,  
    할인 미적용 상품이라면 sale_price만 존재,  
    심지어 상품 데이터가 미완성이라면 buy_price만 있을 수도 있음.

* COALESCE를 쓰면 애플리케이션에서 if/else로 분기할 필요 없이,  
    DB 뷰에서 통일된 effective_price 값을 바로 조회할 수 있습니다.

### 4️⃣ 실제 예시
|product_id | final_price | sale_price | buy_price | effective_price
|---|---|---|---|---|
P001       |   20000     |   25000    |   15000   | 20000   (final_price 사용)
P002       |   NULL      |   30000    |   20000   | 30000   (sale_price 사용)
P003       |   NULL      |   NULL     |   18000   | 18000   (buy_price 사용)
P004       |   NULL      |   NULL     |   NULL    | 0       (모두 NULL → 0)


### ✅ 정리:

* COALESCE(p.final_price, p.sale_price, p.buy_price, 0) 은  
**“상품의 최종적으로 보여줄 가격”**을 결정하는 로직입니다.

* final_price → sale_price → buy_price → 0 순서로 fallback합니다.

* 이렇게 하면 애플리케이션에서 가격표시 시, 항상 effective_price 하나만 사용하면 됩니다