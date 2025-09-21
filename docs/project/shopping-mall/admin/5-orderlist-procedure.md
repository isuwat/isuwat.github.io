---
layout: default
title: "[admin] 주문조회 프로시저"
parent: admin
nav_order: 5
---

# 주문 조회 프로시저 


2) 집계(건수/합계) — 중복 방지용 `EXISTS`

```sql
SELECT
  COUNT(*)                           AS cnt,
  SUM(ods.total_payment)             AS sum_pay
INTO po_total_cnt, po_total_order_price
FROM shop_orders ods
WHERE 1=1
  AND ods.order_date BETWEEN pi_start_date AND pi_end_date
  -- 키워드(회원/성명/전화)
  AND (
    pi_keyword IS NULL OR
    (pi_key_type = '1' AND ods.user_id   = pi_keyword) OR
    (pi_key_type = '2' AND ods.user_name = pi_keyword) OR
    (pi_key_type = '3' AND (ods.user_phone = v_kw_phone OR ods.receiver_phone = v_kw_phone))
  )
  -- 결제수단/상태
  AND (pi_pay_type IS NULL OR pi_pay_type = '' OR ods.pay_type = pi_pay_type)
  AND (pi_order_status IS NULL OR pi_order_status = '' OR ods.order_status = pi_order_status)
  -- 상품명 LIKE: 주문단위 필터는 EXISTS로 (합계 뻥튀기 방지)
  AND (
    pi_product_name IS NULL OR pi_product_name = '' OR
    EXISTS (
      SELECT 1
      FROM shop_order_product op
      WHERE op.order_seq_no = ods.seq_no
        AND op.product_name LIKE CONCAT('%', pi_product_name, '%')
    )
  );

```

**설명**

- **합계/건수는 반드시 `shop_orders` 단독**으로 집계.
    
- 상품명 검색은 **EXISTS**로 “그 상품이 포함된 주문만” 남김 → _중복 합계 방지_.

3) 리스트(표시용) — 요약 서브쿼리 `agg`
```sql
SELECT
  ods.seq_no                AS order_seq_no,
  ods.order_number,
  ods.order_date,
  ods.user_id, ods.user_name, ods.user_phone,
  ods.receiver_phone, ods.receiver_post_code,
  FORMAT(ods.product_total_price, 0) AS product_total_price,
  FORMAT(ods.total_payment, 0)       AS total_payment,
  ods.order_status, ods.pay_type, c.name AS carrier_name,

  -- 표시용 요약
  CASE WHEN agg.item_cnt > 1
       THEN CONCAT(agg.first_name, ' 외 ', agg.item_cnt - 1, '건')
       ELSE agg.first_name
  END AS product_summary,
  agg.product_lines

FROM shop_orders ods
LEFT JOIN shop_couriers c ON c.id = ods.courier_id

-- (표시용) 주문별 상품 요약
LEFT JOIN (
  SELECT
    op.order_seq_no,
    COUNT(*) AS item_cnt,
    SUBSTRING_INDEX(
      GROUP_CONCAT(op.product_name ORDER BY op.reg_date ASC SEPARATOR '||'),
      '||', 1
    ) AS first_name,
    GROUP_CONCAT(
      CONCAT(op.product_name, ' x', op.quantity, '개')
      ORDER BY op.reg_date ASC
      SEPARATOR 0x0A
    ) AS product_lines
  FROM shop_order_product op
  GROUP BY op.order_seq_no
) agg ON agg.order_seq_no = ods.seq_no

WHERE 1=1
  AND ods.order_date BETWEEN pi_start_date AND pi_end_date
  -- 키워드(회원/성명/전화)
  AND (
    pi_keyword IS NULL OR
    (pi_key_type = '1' AND ods.user_id   = pi_keyword) OR
    (pi_key_type = '2' AND ods.user_name = pi_keyword) OR
    (pi_key_type = '3' AND (ods.user_phone = v_kw_phone OR ods.receiver_phone = v_kw_phone))
  )
  -- 결제수단/상태
  AND (pi_pay_type IS NULL OR pi_pay_type = '' OR ods.pay_type = pi_pay_type)
  AND (pi_order_status IS NULL OR pi_order_status = '' OR ods.order_status = pi_order_status)
  -- 상품명 LIKE (주문단위 필터)
  AND (
    pi_product_name IS NULL OR pi_product_name = '' OR
    EXISTS (
      SELECT 1
      FROM shop_order_product op2
      WHERE op2.order_seq_no = ods.seq_no
        AND op2.product_name LIKE CONCAT('%', pi_product_name, '%')
    )
  )

ORDER BY
  CASE WHEN pi_sort = 1 THEN ods.order_date END DESC,
  ods.order_date ASC

LIMIT v_offset, v_limit;

```

**설명**

- `agg`는 **표시 전용**. 리스트 행을 **주문당 1행**으로 만들고,
    
    - `product_summary` = “첫 상품명 외 n건”
        
    - `product_lines` = “상품명 x수량개” 줄바꿈 목록.
        
- 메인 WHERE는 **집계와 동일 조건**을 재사용(결과/합계 일치 보장).
    
- 이미 주문당 1행이라 `GROUP BY`/`ANY_VALUE()` **불필요**.
    

> 선택 옵션: “검색 시 `product_lines`도 매칭 품목만” 보이고 싶다면 `agg` 서브쿼리에 `WHERE op.product_name LIKE ...`를 **조건부**로 주입해도 됨. (미주입 시, 필터는 되지만 줄 요약은 “전체 품목” 표시)

---

# 4) 전화번호 검색 로직(요지)

- **입력**: 하이픈 등 제거 → 숫자만(`v_kw_phone`).
    
- **DB**: `user_phone`, `receiver_phone` 모두 **숫자-only 저장**.
    
- **조건**: `pi_key_type='3'`일 때만 `(user_phone = v_kw_phone OR receiver_phone = v_kw_phone)` 적용.
    

> 프런트에서도 select=3이면 입력 도중 숫자만 유지하면 UX 좋음(이미 적용한 스크립트 OK).

---

#### SUBSTRING_INDEX(), GROUP_CONCAT 는 GROUP BY에서 명시하지 않는 이유?

- `GROUP_CONCAT()`는 **집계 함수**예요. `GROUP BY`로 묶인 **그룹당 1개의 값**을 만들어내므로, `GROUP BY`에 넣는 대상이 아닙니다. (SQL 규칙상 `GROUP BY` 항목에 집계함수는 올 수 없음)
    
- `SUBSTRING_INDEX(GROUP_CONCAT(...), '||', 1)`도 본질적으로 **집계 결과(STRING)**에 대한 **후처리**일 뿐이라, 이것 역시 `GROUP BY`에 명시할 필요가 없습니다. `GROUP BY`에는 “그룹을 나눌 키”만 넣으면 돼요 — 여기선 `op.order_seq_no` 하나로 충분합니다.
    
- 만약 `GROUP BY`에 `product_name` 같은 걸 더 넣으면 “주문당 1행”이 깨지고, **주문이 상품 수만큼 쪼개져** 버립니다(원하는 요약이 무너짐).
    

정리하면:

- 서브쿼리(agg)에서 `GROUP BY op.order_seq_no`만 두고  
    `COUNT(*)`, `GROUP_CONCAT(...)`, `SUBSTRING_INDEX(GROUP_CONCAT(...))`는 **모두 집계식**이므로 `GROUP BY`에 안 넣는 게 맞습니다.
    
- 바깥 쿼리는 이미 **주문당 1행** 구조라 별도 `GROUP BY`가 필요 없고, 그래서 `ANY_VALUE()`도 제거해도 됩니다


WHERE 절은 **각 `shop_orders` 행(= `ods`)에 대해 “이 주문에 이런 상품이 하나라도 있나?”를 검사**합니다.

- `EXISTS ( … WHERE op2.order_seq_no = ods.seq_no AND op2.product_name LIKE … )`가 **참이면 그 주문(ods 행)이 결과에 포함**되고, 거짓이면 제외돼요.
    
- `pi_product_name`이 비어 있으면 앞의 `OR` 때문에 **무조건 통과**합니다.
    

즉, **EXISTS는 출력용이 아니라 필터**예요. 통과된 주문만 SELECT에 “표현”됩니다. (상품명 리스트를 화면에 보여주는 건 별도의 `agg.product_lines` 같은 집계 서브쿼리/컬럼이 담당합니다.)

### 작동 요약(진리표)

- `pi_product_name` = NULL/'' → 조건 전체 TRUE → 주문 **포함**
    
- `pi_product_name` = '키워드' AND 그 주문에 LIKE 매치 상품 ≥ 1 → **포함**
    
- `pi_product_name` = '키워드' AND 매치 없음 → **제외**
    

### 주의 메모

- EXISTS는 **행 존재 여부**만 보니 “매치된 상품명”이 자동으로 표기되진 않습니다. 보기는 `product_lines` 같은 컬럼으로 처리하세요(보통은 주문의 **모든 상품**을 그대로 표시).
    
- 성능 위해 인덱스 권장: `shop_order_product (order_seq_no, product_name)`.
    
- 사용자가 `%`/`_`/`\`를 입력하면 LIKE가 과매치할 수 있으니 이스케이프(`ESCAPE '\\'`)를 고려하세요.