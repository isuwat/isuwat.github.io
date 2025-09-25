---
layout: default
title: "[MySQL] FOR UPDATE"
parent: MySQL
nav_order: 12
---


# FOR UPDATE

# FOR UPDATE

`FOR UPDATE`는 **트랜잭션 안에서 조회된 행을 잠그는(locking) 힌트**입니다.
`SELECT … FOR UPDATE` 는 **조건에 의해 실제 조회되는 행(row)** 만 잠급니다.

### 📌 의미

- `SELECT … FOR UPDATE` 하면, 조회된 행들이 **배타적 잠금(Exclusive Lock)** 상태로 걸립니다.
    
- 같은 트랜잭션이 끝날 때까지(= COMMIT/ROLLBACK 전까지) **다른 트랜잭션이 해당 행을 수정/삭제할 수 없습니다.**
    
- 읽기(`SELECT`)는 가능하지만, 수정(`UPDATE`, `DELETE`)은 블로킹됩니다.
    

---

### 📌 왜 쓰나요?

주문/결제 같은 시나리오에서 **경합 조건(Race Condition)**을 막기 위해서입니다.

예:

1. A, B 사용자가 동시에 같은 장바구니 상품(cart_id=10)을 주문하려 함.
    
2. `SELECT … FOR UPDATE`로 행을 잠그면, 먼저 트랜잭션 잡은 A가 COMMIT하기 전까지  
    B는 cart_id=10을 건드릴 수 없음 → **중복 주문/재고 이슈 방지**
    

---

### 📌 현재 쿼리의 의도
```sql
SELECT 1
FROM shop_cart c
JOIN selected_ids s ON s.cart_id = c.cart_id
WHERE c.user_id = pi_user_id
FOR UPDATE;
```

- `pi_cart_ids_json`으로 전달된 cart_id 들을 **조회하면서 잠금**을 거는 것.
    
- 이후 같은 트랜잭션에서:
    
    - `INSERT INTO shop_order_product …`
        
    - `DELETE FROM shop_cart …`  
        등을 실행할 때, 동시성 문제 없이 안전하게 진행 가능.
        

---

### 📌 참고

- `FOR UPDATE`는 **InnoDB**에서만 동작합니다. (MyISAM은 무시)
    
- 잠금 단위는 **조회된 행(Row Lock)**이며, 인덱스 조건에 따라 **테이블 락**까지 걸릴 수 있음.
    
- 만약 읽기만 하고 수정/삭제 계획이 없다면 `LOCK IN SHARE MODE`(공유락)도 선택 가능하지만, 주문 시나리오에선 보통 `FOR UPDATE`.
    

---

### 📌 동작 예시

- `shop_cart` 테이블에 데이터:
```sql
cart_id | user_id | product_id | quantity
-----------------------------------------
10      | A       | 101        | 1
11      | B       | 102        | 2
12      | A       | 103        | 1

```

- 프로시저 호출: `pi_user_id = 'A', pi_cart_ids_json = [10,12]`
    
- 잠금 범위:
    
    - cart_id = 10 (user_id = A)
        
    - cart_id = 12 (user_id = A)
        
- → **행 10, 12만 잠금**
    
- 행 11(B의 장바구니)은 영향 없음
    

---

### 📌 주의점

- `WHERE` 조건 + `JOIN` 조건에 해당하는 **행만 잠김**
    
- 하지만 **인덱스가 없으면** InnoDB 옵티마이저가 범위 락(table-level gap lock, next-key lock 등)을 더 넓게 잡을 수 있음 → 불필요한 경합 가능
    
- 그래서 `shop_cart.user_id`, `shop_cart.cart_id` 에 **복합 인덱스**가 있으면 안전하고 효율적인 행 잠금만 발생합니다

👉 결론:  
`FOR UPDATE`는 **“이 장바구니 상품들을 내가 주문 처리할 때까지 다른 트랜잭션이 수정하지 못하게 잠가둔다”**는 의미입니다.