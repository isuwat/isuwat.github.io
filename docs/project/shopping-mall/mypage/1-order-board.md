---
layout: default
title: "[mypage] 나의 주문처리현황 집계"
parent: mypage
nav_order: 1
---


# 나의 주문처리 현황 집계

```sql
/* 나의 주문처리현황 집계 */
SELECT
    /* 결제완료 */
    COALESCE(SUM(CASE WHEN o.order_status = 'paid' THEN 1 ELSE 0 END), 0)          AS paid_cnt,
    /* 배송준비중 */
    COALESCE(SUM(CASE WHEN o.order_status = 'preparing' THEN 1 ELSE 0 END), 0)     AS preparing_cnt,
    /* 배송중 */
    COALESCE(SUM(CASE WHEN o.order_status = 'shipping' THEN 1 ELSE 0 END), 0)      AS shipping_cnt,
    /* 배송완료 (과거 complete 포함 운용 시 함께 집계) */
    COALESCE(SUM(CASE WHEN o.order_status IN ('delivered','complete') THEN 1 ELSE 0 END), 0) AS delivered_cnt
INTO o_paid, o_preparing, o_shipping, o_delivered
FROM shop_orders o
WHERE o.user_id = p_user_id;   -- 필요 시 조건 유지
```



