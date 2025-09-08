---
layout: default
title: "[cart] 할인율"
parent: shop
nav_order: 25
---



# 할인율

`v_products_effective` 뷰에서 `effective_price`는 단순히 `final_price → sale_price → buy_price` 중 하나를 쓰고 있는데, 여기에 **할인율(`discount_rate`)** 을 반영해 “실제 결제 금액”을 미리 계산해서 `final_price` 컬럼으로 내려주고 싶으신 거군요.

---

## 1. 할인율 컬럼 구조

- `shop_products.discount_rate` 값이 **0~1 사이 비율**이라고 가정할게요.  
    (예: `0.15` → 15% 할인)
    
- 만약 `0~100` 정수라면 계산식만 약간 조정하면 됩니다.
    

---

## 2. 할인 반영한 컬럼 추가

뷰 정의를 이렇게 수정할 수 있습니다:

```sql
CREATE OR REPLACE
    ALGORITHM = MERGE 
    SQL SECURITY DEFINER
VIEW `v_products_effective` AS
SELECT 
    p.product_id           AS product_id,
    p.product_name         AS product_name,
    COALESCE(p.final_price, p.sale_price, p.buy_price, 0) AS effective_price,
    -- 할인 적용 최종가
    ROUND(
        COALESCE(p.final_price, p.sale_price, p.buy_price, 0) 
        * (1 - COALESCE(p.discount_rate, 0))
    ) AS final_price,
    p.product_thumb_img_url AS image_url,
    p.display              AS display,
    p.stat                 AS stat,
    COALESCE(p.quantity_stock, 0) AS quantity_stock
FROM shop_products p
WHERE p.display = '1'
  AND p.stat <> '2'
  AND COALESCE(p.quantity_stock, 0) > 0;

```

---

## 3. 결과 컬럼 설명

- `effective_price` : 기존 로직 그대로 (할인 전 기준가)
    
- `final_price` : 할인율 반영된 값 (소수점은 `ROUND` 처리)
    

---

## 4. 예시

|sale_price|discount_rate|effective_price|final_price|
|---|---|---|---|
|10,000|0.15|10000|8500|
|50,000|0.30|50000|35000|
|30,000|NULL|30000|30000|

---

✅ 정리  
뷰에서 `final_price` 컬럼을 추가하려면, 할인율까지 계산하는 식을 직접 넣으시면 됩니다.

- `* (1 - discount_rate)` 방식
    
- 소수점은 `ROUND`/`FLOOR`/`CEIL` 중 정책에 맞게 선택


### `final_price` 칼럼 제거 후

- 테이블에 `sale_price`만 둔다면,

```sql
COALESCE(p.sale_price, 0) AS effective_price
```
    
    또는 그냥
```sql
p.sale_price AS effective_price
```
       
    로 충분합니다.
    
- 거기에 할인율(`discount_rate`) 반영을 원한다면, 아예 계산식으로:

```sql
ROUND(p.sale_price * (1 - COALESCE(p.discount_rate, 0))) AS final_price
```
