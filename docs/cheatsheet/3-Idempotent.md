---
layout: default
title: 멱등(Idempotent)
parent: Cheat Sheet
nav_order: 3
---

# 멱등


"멱등(Idempotent)"은 **동일한 연산을 여러 번 수행해도 최종 결과가 변하지 않는 성질**을 말합니다.

- **수학**: f(f(x)) = f(x) 같은 함수가 멱등 함수입니다. 예: 절댓값 함수 |x|는 여러 번 적용해도 결과가 동일합니다.
    
- **HTTP/REST API**:
    
    - `GET /orders/123` → 몇 번 호출해도 결과는 동일 (조회는 멱등).
        
    - `DELETE /cart/5` → 여러 번 호출해도 결국 장바구니에서 5번 아이템은 삭제된 상태 (멱등).
        
    - `POST /orders` → 매번 새로운 주문이 생성되므로 멱등이 아님.
        
- **DB/트랜잭션**: `UPDATE orders SET status='paid' WHERE id=1` 같은 쿼리는 여러 번 실행해도 상태가 'paid'인 것 하나로 유지 → 멱등.  
    하지만 `INSERT INTO ...`는 같은 데이터를 중복 삽입할 수 있어 멱등이 아님.
    

👉 요약하면:  
멱등은 **"중복 실행이 발생해도 의도한 최종 상태가 변하지 않도록 보장하는 성질"** 입니다.