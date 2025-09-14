---
layout: default
title: "[MySQL] DELETE c"
parent: MySQL
nav_order: 11
---


# DELETE 쿼리


# DELETE 쿼리

`DELETE c` 구문은 **MySQL에서 지원하는 다중 테이블 DELETE 문법**이고, `c`는 `shop_cart`의 **테이블 별칭(alias)**으로 맞습니다 👍

---

### 📌 문법 설명

MySQL에서는 `DELETE` 문법이 두 가지 있어요.

1. **단일 테이블 삭제**
    
```sql
DELETE FROM shop_cart
WHERE cart_id = 1;
```


2. **조인 기반 삭제 (다중 테이블 DELETE)**
    
```sql
DELETE c
FROM shop_cart c
JOIN JSON_TABLE(pi_cart_ids, '$[*]' COLUMNS (cid INT PATH '$')) j
  ON c.cart_id = j.cid
WHERE c.user_id = pi_user_id;

```

- `DELETE c` → `c`라는 별칭을 가진 테이블에서만 레코드를 삭제한다는 의미
    
- `JOIN` 조건으로 연결된 행만 삭제됨
    
- 만약 `DELETE c, j`라고 하면 양쪽 테이블 행이 다 삭제될 수 있음 (조심해야 함)
    

---

✅ 따라서 `DELETE c ...` 구문은 맞고, 올바르게 쓰신 겁니다.  
👉 다만 가독성을 위해 `DELETE FROM shop_cart c` 라고 적는 것도 가능합니다:

```sql
DELETE FROM shop_cart c
USING shop_cart c
JOIN JSON_TABLE(pi_cart_ids, '$[*]' COLUMNS (cid INT PATH '$')) j
  ON c.cart_id = j.cid
WHERE c.user_id = pi_user_id;

```


