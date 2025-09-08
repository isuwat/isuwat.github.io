---
layout: default
title: "[cart] 배송비"
parent: shop
nav_order: 19
---


# 배송비


`grouped AS (...)`는 **공통 테이블 표현식(CTE)**으로, 배송비 계산을 위해 장바구니 아이템들을 배송 정책 단위로 묶은 단계입니다.

---

### 코드 원문

```sql
grouped AS (
    SELECT
        delivery_policy_id,
        type,
        base_cost,
        free_over,
        combine_group_id,
        SUM(effective_price * quantity) AS group_subtotal
    FROM cart_items
    GROUP BY delivery_policy_id, type, base_cost, free_over, combine_group_id
)

```

---

### 각 컬럼 의미

- `delivery_policy_id` : 상품에 연결된 배송 정책 ID (예: 무료/조건부/고정 배송비 정책 구분)
    
- `type` : 배송 정책 타입 (예: `'free'`, `'fixed'`, `'conditional'`, `'area'`)
    
- `base_cost` : 정책에서 정의한 기본 배송비 금액
    
- `free_over` : 일정 금액 이상이면 배송비 무료가 되는 조건 금액
    
- `combine_group_id` : 묶음 배송 그룹 ID (같은 그룹끼리는 배송비 1회만 부과)
    
- `SUM(effective_price * quantity) AS group_subtotal`
    
    - 그룹별 상품들의 소계 금액
        
    - 예: 같은 정책/묶음 그룹에 속한 상품들을 전부 합산한 금액
        

---

### 왜 GROUP BY 하는가?

- 장바구니에는 여러 상품이 담길 수 있고, 상품마다 배송 정책이 다를 수 있음.
    
- 정책별로 묶어서 **배송비를 그룹 단위로 계산**해야 함.
    
    - 예:
        
        - A상품: 10,000원, 정책ID=1, 조건부 무료 (30,000 이상 무료, 기본 3,000원)
            
        - B상품: 25,000원, 정책ID=1, 같은 조건부 무료
            
        - → 같은 정책이므로 group_subtotal = 35,000 → 배송비 0원
            
    - C상품: 5,000원, 정책ID=2, 고정배송비 2,500원
        
        - → 따로 그룹핑되어 배송비 2,500원 발생
            

---

### 이후 단계에서 어떻게 쓰이나?

`shipping_calc` CTE에서:
```sql
SUM(
  CASE
    WHEN type = 'free' THEN 0
    WHEN type = 'fixed' THEN base_cost
    WHEN type = 'conditional' THEN
         CASE
           WHEN group_subtotal >= free_over THEN 0
           ELSE base_cost
         END
    ...
  END
) AS shipping_total

```


즉, `grouped`에서 만든 **group_subtotal**과 정책 정보를 바탕으로 배송비 총합을 산출합니다.

---

✅ 정리

- `grouped`는 장바구니 상품들을 **배송 정책 단위로 묶은 중간 집계 테이블**입니다.
    
- 같은 배송 정책/묶음 그룹에 속한 상품들은 하나의 그룹으로 합쳐지고, 그 소계를 `group_subtotal`로 계산합니다.
    
- 이후 단계에서 이 값으로 **조건부 무료 배송, 고정 배송비 적용** 등을 판별합니다.


SQL의 `GROUP BY`는 지정한 컬럼들의 **모든 값이 동일한 행**들만 하나의 그룹으로 묶습니다.

---

### 현재 구문

`GROUP BY delivery_policy_id, type, base_cost, free_over, combine_group_id`

- 즉, 다섯 개 컬럼(`delivery_policy_id`, `type`, `base_cost`, `free_over`, `combine_group_id`)의 값이 **모두 같아야 같은 그룹**이 됩니다.
    

---

### 예시

#### cart_items (예시 데이터)

|product_id|delivery_policy_id|type|base_cost|free_over|combine_group_id|effective_price|quantity|
|---|---|---|---|---|---|---|---|
|A|1|conditional|3000|30000|10|10000|1|
|B|1|conditional|3000|30000|10|20000|1|
|C|2|fixed|2500|NULL|20|5000|2|
|D|1|conditional|3000|20000|10|15000|1|

---

#### grouped 결과

- 상품 A와 B → **전부 값이 동일** (`delivery_policy_id=1`, `type=conditional`, `base_cost=3000`, `free_over=30000`, `combine_group_id=10`)  
    → 같은 그룹으로 합쳐져 `group_subtotal = 10000*1 + 20000*1 = 30000`
    
- 상품 C → 정책 ID 2, type=fixed → 별도 그룹  
    → `group_subtotal = 5000*2 = 10000`
    
- 상품 D → 정책 ID는 같지만, `free_over`가 다름 (20000 vs 30000)  
    → A, B와는 다른 그룹으로 분리됨  
    → `group_subtotal = 15000`
    

---

### 결론

`GROUP BY delivery_policy_id, type, base_cost, free_over, combine_group_id`  
👉 **다섯 필드 값이 모두 일치해야만** 같은 그룹으로 묶입니다.  
하나라도 다르면 별도 그룹이 됩니다.