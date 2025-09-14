---
layout: default
title: "[MySQL] coalesce(nullif())"
parent: MySQL
nav_order: 10
---


# COALESCE(NULLIF())



# COALESCE(NULLIF())

`COALESCE(NULLIF(p.sale_price, 0), p.buy_price)` 는 MySQL에서 자주 쓰는 **Fallback(대체값) 패턴**이에요.

순서대로 풀어보면:

1. **`NULLIF(p.sale_price, 0)`**
    
    - `p.sale_price`가 `0`이면 `NULL`을 반환합니다.
        
    - `p.sale_price`가 `0`이 아니면 원래 값을 그대로 반환합니다.  
        👉 즉, “0이면 버려라(=NULL 취급)”.
        
2. **`COALESCE(... , p.buy_price)`**
    
    - 첫 번째 인자(`NULLIF(...)`)가 `NULL`이 아니면 그 값을 반환합니다.
        
    - `NULL`이면 두 번째 인자 `p.buy_price`를 반환합니다.

`COALESCE(expr1, expr2, expr3, …)` 는 **인자들을 왼쪽부터 차례대로 검사해서, NULL이 아닌 첫 번째 값을 반환**합니다.

---

### 동작 방식

1. `expr1`이 **NULL이 아니면** → `expr1` 반환
    
2. `expr1`이 **NULL이면** → `expr2` 검사
    
3. `expr2`가 **NULL이 아니면** → `expr2` 반환
    
4. 모두 NULL이면 최종적으로 NULL 반환
    

즉, **NULL 여부만 체크**하지, 0·빈문자열·FALSE 같은 값들은 그대로 “NULL 아님”으로 취급합니다.

```sql
SELECT COALESCE(NULL, NULL, 'Hello', 'World');  -- 결과: 'Hello'
SELECT COALESCE(0, 100);                        -- 결과: 0   (0은 NULL 아님)
SELECT COALESCE('', 'fallback');                -- 결과: ''  (빈 문자열은 NULL 아님)
SELECT COALESCE(NULL, 5, 10);                   -- 결과: 5

```

👉 그래서 `COALESCE(NULLIF(p.sale_price, 0), p.buy_price)` 같은 패턴이 유용한 겁니다.

- 그냥 `COALESCE(p.sale_price, p.buy_price)`라면 0을 NULL로 보지 않기 때문에 **0 그대로 반환**해버려요.
    
- 하지만 `NULLIF(p.sale_price, 0)`로 먼저 0을 NULL로 바꿔주면 → `COALESCE`가 `buy_price`로 fallback 하게 되는 거죠.
    

---

정리:

- `COALESCE()`는 오직 “NULL 여부”만 보고 선택합니다.
    
- 그래서 0이나 공백 등을 “무효값”으로 취급하려면 `NULLIF`나 `CASE`랑 조합해서 써야 합니다.

---

### 따라서 전체 의미는:

- `sale_price`가 0이거나 NULL → `buy_price` 사용
    
- `sale_price`가 0이 아닌 정상값 → 그 값을 사용
    

---

✔ 예시

|sale_price|buy_price|결과|
|---|---|---|
|10000|8000|10000|
|0|8000|8000|
|NULL|8000|8000|
즉, 이 구문은 **“판매가가 없거나 0이면 매입가를 대신 써라”** 라는 로직을 아주 간결하게 표현한 것입니다.

`NULLIF(expr1, expr2)` 는 **expr1과 expr2가 같으면 NULL, 다르면 expr1**을 반환하는 함수입니다.

---

### `NULLIF(p.sale_price, 1)` 의 의미

- 만약 `p.sale_price = 1` → `NULL` 반환
    
- 만약 `p.sale_price ≠ 1` → `p.sale_price` 반환
    

---

✔ 예시

|p.sale_price|결과 (NULLIF(p.sale_price, 1))|
|---|---|
|1|NULL|
|0|0|
|5000|5000|
|NULL|NULL|

---

즉, **특정 값(여기서는 1)을 “없음(NULL)”으로 취급**하고 싶을 때 쓰는 함수예요.  
보통 `NULLIF(p.sale_price, 0)`처럼 “0이면 무효값으로 치자”라는 로직에서 많이 쓰고, 다른 값(예: 1, -1, 'N/A')도 마찬가지로 쓸 수 있습니다.

---

👉 정리:

- `NULLIF(p.sale_price, 0)` → 0이면 NULL 취급
    
- `NULLIF(p.sale_price, 1)` → 1이면 NULL 취급

##### COALESCE(`p`.`quantity_stock`, 0) AS `quantity_stock`, <- 이건 stock이 0이면?

`COALESCE(p.quantity_stock, 0)` 은 **stock 컬럼이 `NULL`일 때만 0으로 대체**합니다.  
즉, 실제 값이 **0**이면 그대로 **0**이 반환돼요.

---

✔ 케이스별 결과

|p.quantity_stock|결과|
|---|---|
|NULL|0|
|0|0|
|5|5|
|100|100|

---

👉 따라서 이 표현식은 **“재고가 기록되어 있지 않으면(=NULL) 0으로 보정”** 이라는 의미고,  
실제로 재고가 0인 경우는 **그대로 0**으로 유지됩니다.

---

✅ 만약 “0도 무효값으로 보고 다른 값으로 바꿔야” 한다면,  
`NULLIF`를 같이 써야 합니다.
예:
```sql
COALESCE(NULLIF(p.quantity_stock, 0), -1) AS quantity_stock
```

→ 재고가 `0`이면 `NULL`로 변환되고, 그럼 `-1`로 fallback.