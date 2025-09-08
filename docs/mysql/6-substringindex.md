---
layout: default
title: "[MySQL] SUBSTRING_INDEX"
parent: MySQL
nav_order: 6 
---

# SUBSTRING_INDEX

### MySQL
`SUBSTRING_INDEX(GROUP_CONCAT(...), ',', 1)` 같은 패턴은 **GROUP BY로 묶은 그룹 안에서 대표값 하나 뽑고 싶을 때** 쓰는 방법이에요.

## 원리 설명

1. **`GROUP_CONCAT(expr ORDER BY … SEPARATOR ',')`**
    
    - 그룹 내 행들을 지정한 순서대로 **문자열로 이어붙입니다**.
        
    - 예: 상품명이 `[사과, 배, 감]` 이고 ORDER BY로 `사과, 배, 감` 순으로 지정하면 결과는
        
        `"사과,배,감"`
        
2. **`SUBSTRING_INDEX(str, delim, count)`**
    
    - `str` 문자열을 `delim` 구분자로 잘라서 **앞에서 count번째까지** 반환합니다.
        
    - `count > 0` → 왼쪽부터 n번째 구분자까지
        
    - `count < 0` → 오른쪽부터 n번째 구분자까지
        
    
    예:
```sql
SELECT SUBSTRING_INDEX('사과,배,감', ',', 1); -- '사과' 
SELECT SUBSTRING_INDEX('사과,배,감', ',', 2); -- '사과,배' 
SELECT SUBSTRING_INDEX('사과,배,감', ',', -1); -- '감'
```
    
    
3. 따라서 `SUBSTRING_INDEX(GROUP_CONCAT(...), ',', 1)`  
    → 그룹 내에서 정렬된 문자열의 **첫 번째 값**만 뽑아냅니다.
    

---

## 실무 활용 예

- **대표 상품 이름**:
```sql
SUBSTRING_INDEX(GROUP_CONCAT(p.product_name ORDER BY op.seq_no DESC), ',', 1)
```
    
    
- **대표 이미지**:

```sql
SUBSTRING_INDEX(GROUP_CONCAT(p.product_thumb_img_url ORDER BY op.seq_no DESC), ',', 1)
```
    
    
- **대표 옵션명**: 동일 방식.
    

즉, `GROUP_CONCAT`으로 “모든 값”을 한 줄 문자열로 만든 다음,  
`SUBSTRING_INDEX`로 **첫 번째만 슬라이스**해서 대표값으로 씁니다.

---

✅ 정리

- `GROUP_CONCAT` → 그룹 내 여러 행을 문자열로 합친다.
    
- `SUBSTRING_INDEX(..., ',', 1)` → 그 중 맨 앞의 값 하나만 뽑는다.
    
- ORDER BY 옵션을 잘 쓰면 **“정렬 기준에 따라 대표 1건 뽑기”**가 가능합니다.

### 🔎 왜 필요한가?

보통 `GROUP BY`를 하면 **집계 함수(SUM, COUNT, MAX …)**만 쓸 수 있지, 그냥 일반 컬럼을 SELECT에 넣으면 에러가 나거나 임의의 값이 나옵니다.  
→ 그래서 "여러 행 중 대표값 하나"를 뽑아야 할 때 트릭으로 `GROUP_CONCAT + SUBSTRING_INDEX`를 씁니다.



### 📌 동작 흐름

예를 들어 주문에 여러 상품이 있을 때:

|order_no|product_name|
|---|---|
|1001|사과|
|1001|배|
|1001|감|
```sql
SELECT    order_no,   SUBSTRING_INDEX(GROUP_CONCAT(product_name ORDER BY product_name ASC), ',', 1) AS first_item FROM order_items GROUP BY order_no;
```


결과:

|order_no|first_item|
|---|---|
|1001|감|

(`ORDER BY product_name ASC` 기준으로 가장 작은 "감"이 대표로 뽑힘)

---

### 📌 일반 조회(그룹 없는 경우)

- 그냥 한 행만 조회할 때는 필요 없습니다.
    
- `GROUP_CONCAT` 자체가 **여러 행 → 한 문자열**을 만들기 위한 집계 함수라, GROUP BY가 없으면 전체 테이블을 하나의 그룹으로 취급합니다.
    

예:
```sql
SELECT GROUP_CONCAT(product_name) FROM order_items;
```


→ 모든 상품명이 `"사과,배,감"` 같이 한 줄로 합쳐져 나옵니다.

---

✅ 정리

- **여러 행을 그룹핑했을 때**: `GROUP_CONCAT`으로 합치고, `SUBSTRING_INDEX`로 첫 값만 추출 → 대표값 뽑기.
    
- **그룹핑 안 하고 전체에서 한 값만**: 그냥 `LIMIT 1`, `ORDER BY` 쓰는 게 더 간단합니다.


## 예시 데이터

`shop_orders` / `shop_order_product` / `shop_products`를 단순화해서:

**shop_orders**

|seq_no|order_number|order_date|order_status|total_payment|
|---|---|---|---|---|
|1|ORD001|2025-09-01|paid|15000|

**shop_order_product**

|order_seq_no|product_id|quantity|
|---|---|---|
|1|101|2|
|1|102|1|
|1|103|3|

**shop_products**

|product_id|product_name|
|---|---|
|101|사과|
|102|배|
|103|감|

---

## 실행 쿼리

```sql
SELECT
    o.seq_no,
    o.order_number,
    o.order_date,
    o.order_status,
    o.total_payment,

    /* 콤팩트: 대표 상품(첫 행) + 총 수량 */
    SUBSTRING_INDEX(
        GROUP_CONCAT(
            p.product_name
            ORDER BY COALESCE(op.seq_no, op.product_id) DESC
            SEPARATOR ','
        ),
        ',',
        1
    ) AS first_item_name,

    SUM(op.quantity) AS total_item_count

FROM shop_orders o
JOIN shop_order_product op ON op.order_seq_no = o.seq_no
JOIN shop_products p       ON p.product_id = op.product_id
WHERE o.seq_no = 1
GROUP BY o.seq_no, o.order_number, o.order_date, o.order_status, o.total_payment;

```


---

## 결과

|seq_no|order_number|order_date|order_status|total_payment|first_item_name|total_item_count|
|---|---|---|---|---|---|---|
|1|ORD001|2025-09-01|paid|15000|감|6|

- `GROUP_CONCAT(p.product_name …)` → `"감,배,사과"` (정렬 기준 때문에 이렇게 나왔다고 가정)
    
- `SUBSTRING_INDEX(..., ',', 1)` → `"감"` → **대표 상품명**
    
- `SUM(op.quantity)` → 2+1+3 = 6 → **총 수량**
    

---

✅ 요약  
이 구문은 `GROUP_CONCAT`으로 **모든 상품명을 하나의 문자열**로 만들고,  
`SUBSTRING_INDEX(..., ',', 1)`로 **첫 번째 값 하나만 잘라서** “대표 상품”을 선택하는 패턴입니다.


##  `GROUP BY`에 왜 저렇게 여러 컬럼을 다 넣는지 설명드릴게요.

---

## 1. SQL 표준 규칙

- `GROUP BY`를 쓰면, **집계 함수( SUM, COUNT, GROUP_CONCAT … )가 아닌 일반 컬럼**은  
    **GROUP BY 절에 반드시 포함**되어야 합니다.
    
- 그렇지 않으면 MySQL은 모드에 따라 에러(`ONLY_FULL_GROUP_BY`)를 내거나, 임의의 값(비권장)을 반환합니다.
    

즉, `SELECT`절에 있는 `o.order_number`, `o.order_date` 같은 컬럼을 그대로 출력하고 싶으면  
→ `GROUP BY`에도 똑같이 써야 합니다.

---

## 2. 왜 이렇게 여러 개?

`GROUP BY o.seq_no` 하나만 써도 사실상 **주문 한 건씩** 그룹핑이 됩니다.  
왜냐하면 `seq_no`가 **PK(고유 키)**라서 주문별로 유일하기 때문이죠.

그런데도 `order_number`, `order_date`, `order_status`, `total_payment`까지 같이 넣은 이유는:

1. **표준 SQL 호환성/가독성**  
    → SELECT에 나온 모든 non-aggregate 컬럼을 GROUP BY에 다 써주면  
    DB 모드/버전 상관없이 안전합니다.
    
2. **여러 컬럼 조합이 진짜 그룹 기준이 될 수도 있어서**  
    (예: seq_no가 PK가 아닌 환경을 상정하거나, 추후 조인/쿼리 재사용 고려)
    

---

## 3. 실무에서의 선택

- **간단히 하려면**:
    
    `GROUP BY o.seq_no`
    
    → PK로 그룹핑하면 충분.
    
- **안전하게, 에러 피하려면**:
    
    `GROUP BY o.seq_no, o.order_number, o.order_date, o.order_status, o.total_payment`
    
    → SELECT에 나온 컬럼을 전부 포함.
    

---

✅ 정리

- `GROUP BY`에 모든 컬럼을 넣은 건 **SQL 표준/안전성** 때문에.
    
- 하지만 주문번호(`seq_no`)가 PK라면 `GROUP BY o.seq_no` 하나만으로도 충분히 동일한 결과가 나옵니다.