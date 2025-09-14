---
layout: default
title: "[order] 최종 결제 금액 프로시저"
parent: order
nav_order: 5
---




# calc_cart_totals

```sql
DROP PROCEDURE IF EXISTS calc_cart_totals;
DELIMITER $$

CREATE PROCEDURE calc_cart_totals(
    IN  pi_user_id VARCHAR(20),
    IN  pi_selected_cart_ids JSON,
    OUT po_subtotal BIGINT UNSIGNED,
    OUT po_shipping BIGINT UNSIGNED,
    OUT po_discount BIGINT UNSIGNED,
    OUT po_total    BIGINT UNSIGNED
)
BEGIN
    DECLARE v_subtotal BIGINT UNSIGNED DEFAULT 0;
    DECLARE v_shipping BIGINT UNSIGNED DEFAULT 0;
    DECLARE v_discount BIGINT UNSIGNED DEFAULT 0;
    DECLARE v_total    BIGINT UNSIGNED DEFAULT 0;

    /* 1) 상품 소계 */
    SELECT COALESCE(SUM(li.quantity * li.final_price), 0)
      INTO v_subtotal
    FROM (
        SELECT c.seq_no, c.quantity, v.final_price
        FROM shop_cart c
        JOIN JSON_TABLE(pi_selected_cart_ids, '$[*]'
               COLUMNS (cid BIGINT PATH '$.id')) j
          ON j.cid = c.seq_no
        JOIN v_products_effective v
          ON v.product_id = c.product_id
        WHERE c.user_id = pi_user_id
    ) li;

    /* 2) 배송비 */
    SELECT COALESCE(SUM(
        CASE
            WHEN g.type = 'free'        THEN 0
            WHEN g.type = 'fixed'       THEN g.base_cost
            WHEN g.type = 'conditional' THEN
                CASE
                    WHEN g.group_subtotal = 0 THEN 0
                    WHEN g.free_over IS NOT NULL AND g.group_subtotal >= g.free_over THEN 0
                    ELSE g.base_cost
                END
            WHEN g.type = 'area'        THEN g.base_cost
            ELSE 0
        END
    ), 0)
      INTO v_shipping
    FROM (
        SELECT
            dp.id                    AS delivery_policy_id,
            dp.type                  AS type,
            dp.base_cost             AS base_cost,
            dp.free_over             AS free_over,
            COALESCE(dp.combine_group_id, -1) AS combine_group_id,
            SUM(v.final_price * c.quantity)   AS group_subtotal
        FROM shop_cart c
        JOIN JSON_TABLE(pi_selected_cart_ids, '$[*]'
               COLUMNS (cid BIGINT PATH '$.id')) j
          ON j.cid = c.seq_no
        JOIN v_products_effective v ON v.product_id = c.product_id
        JOIN shop_products p        ON p.product_id = c.product_id
        LEFT JOIN shop_delivery_policy dp ON dp.id = p.delivery_policy_id
        WHERE c.user_id = pi_user_id
        GROUP BY dp.id, dp.type, dp.base_cost, dp.free_over, COALESCE(dp.combine_group_id,-1)
    ) g;

    /* 3) 할인 & 총계 */
    SET v_discount := 0;
    SET v_total    := v_subtotal + v_shipping - v_discount;

    /* OUT으로 반환 */
    SET po_subtotal = v_subtotal;
    SET po_shipping = v_shipping;
    SET po_discount = v_discount;
    SET po_total    = v_total;
END$$

DELIMITER ;

```

#### 주의 : selected_cart_ids json 형태

```sql
-- 공통 CTE: selected_ids (객체배열과 숫자배열을 모두 지원)
WITH selected_ids AS (
  -- case 1: [{"id": 25379}, ...]
  SELECT CAST(j1.cid AS UNSIGNED) AS cart_seq
  FROM JSON_TABLE(pi_selected_cart_ids, '$[*]'
         COLUMNS (cid BIGINT PATH '$.id')) AS j1
  WHERE j1.cid IS NOT NULL
  UNION ALL
  -- case 2: [25379, 25380, ...]
  SELECT CAST(j2.cid AS UNSIGNED) AS cart_seq
  FROM JSON_TABLE(pi_selected_cart_ids, '$[*]'
         COLUMNS (cid BIGINT PATH '$')) AS j2
  WHERE j2.cid IS NOT NULL
)
```
