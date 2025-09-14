---
layout: default
title: "[order] 상품가격 fallback"
parent: order
nav_order: 3
---



# 상품가격 fallback

## 🎯 v_products_effective 뷰 설계 원칙

- **중복 조인 제거**  
    → 원래 `shop_products`와 뷰를 동시에 조인했는데, 뷰 안에 필요한 컬럼(`category_id`, `discount_rate`)을 포함시켜 **뷰 단독 사용** 가능하도록 변경.
    
- **판매가/할인가 계산 일원화**  
    → 모든 가격 계산 로직을 뷰 안에서 처리 → 조회 쿼리에서는 단순 SELECT만.
    
- **Fallback 처리**  
    → `sale_price`가 0이면 `buy_price`를 대신 사용. (`COALESCE(NULLIF(...))` 패턴)
    
- **NULL 안전성 확보**  
    → 할인율(`discount_rate`)·재고(`quantity_stock`) 등에 `COALESCE` 사용.
    
- **가공 컬럼 제공**  
    → 화면/로직에서 바로 쓰도록 `final_price`, `dis_rate`, `dis_price` 등을 미리 계산.

```sql
CREATE 
    ALGORITHM = MERGE 
    SQL SECURITY DEFINER
VIEW v_products_effective AS
SELECT 
    p.product_id,
    p.product_name,

    -- 판매가: 0이면 매입가 fallback
    COALESCE(NULLIF(p.sale_price, 0), p.buy_price) AS sale_price,

    -- 최종가: 할인율 적용
    ROUND(
        COALESCE(NULLIF(p.sale_price, 0), p.buy_price) 
        * (1 - COALESCE(p.discount_rate, 0)), 0
    ) AS final_price,

    -- 할인율 %
    CAST(ROUND(COALESCE(p.discount_rate, 0) * 100, 0) AS UNSIGNED) AS dis_rate,

    -- 할인금액
    COALESCE(NULLIF(p.sale_price, 0), p.buy_price) 
      - ROUND(
            COALESCE(NULLIF(p.sale_price, 0), p.buy_price) 
            * (1 - COALESCE(p.discount_rate, 0)), 0
        ) AS dis_price,

    p.product_thumb_img_url AS image_url,
    p.display,
    p.stat,
    COALESCE(p.quantity_stock, 0) AS quantity_stock,
    p.category_id,
    p.discount_rate
FROM shop_products p
WHERE p.display = '1'
  AND p.stat <> '2'
  AND COALESCE(p.quantity_stock, 0) > 0;

```

🔑 **장점**    
- `sale_price`, `final_price`, `dis_price`가 항상 동일한 기준(`COALESCE(NULLIF(...))`)을 사용


`COALESCE(NULLIF(p.sale_price, 0), p.buy_price)` 는 MySQL에서 자주 쓰는 **Fallback(대체값) 패턴**이에요.

순서대로 풀어보면:

1. **`NULLIF(p.sale_price, 0)`**
    
    - `p.sale_price`가 `0`이면 `NULL`을 반환합니다.
        
    - `p.sale_price`가 `0`이 아니면 원래 값을 그대로 반환합니다.  
        👉 즉, “0이면 버려라(=NULL 취급)”.
        
2. **`COALESCE(... , p.buy_price)`**
    
    - 첫 번째 인자(`NULLIF(...)`)가 `NULL`이 아니면 그 값을 반환합니다.
        
    - `NULL`이면 두 번째 인자 `p.buy_price`를 반환합니다.
        

---

### 따라서 전체 의미는:

- `sale_price`가 0이거나 NULL → `buy_price` 사용
    
- `sale_price`가 0이 아닌 정상값 → 그 값을 사용

✔ 예시

|sale_price|buy_price|결과|
|---|---|---|
|10000|8000|10000|
|0|8000|8000|
|NULL|8000|8000|

---

즉, 이 구문은 **“판매가가 없거나 0이면 매입가를 대신 써라”** 라는 로직을 아주 간결하게 표현한 것입니다.

## ✅ 주요 특징

1. **sale_price fallback 처리**:
    
    - `COALESCE(NULLIF(p.sale_price, 0), p.buy_price)`
        
    - → 판매가가 없거나 0이면 매입가 사용.
        
2. **계산식 단순화**:
    
    - `final_price`, `dis_rate`, `dis_price` 모두 `base_price`를 기준으로 계산.
        
    - 로직 중복 제거.
        
3. **데이터 무결성 확보**:
    
    - `quantity_stock`이 `NULL`이면 0으로.
        
    - `discount_rate`가 `NULL`이면 0% 할인 처리.
        
4. **쿼리 간소화**:
    
    - 외부에서 `category_id`, `sale_price`, `final_price`, `dis_price`, `dis_rate` 바로 조회 가능.
        
    - 불필요한 `shop_products` 조인 제거.
