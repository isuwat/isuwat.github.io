---
layout: default
title: "[order] 계산로직 일원화"
parent: order
nav_order: 8
---


# 계산로직 일원화

👉 “여러 곳에서 중복되던 계산/조건을 **한 군데(뷰)**에 모아두고, 외부에서는 그대로 가져다 쓰게 하는 것”을 의미합니다.

---

### 🔎 왜 일원화가 필요할까?

기존에는…

- 상품 목록 쿼리에서 `sale_price`, `final_price`, `dis_rate`, `dis_price`를 **매번 직접 계산**해야 했습니다.
    
- 다른 페이지(상품 상세, 장바구니, 주문서)에서도 똑같은 계산식이 들어가다 보니 **로직 중복 → 유지보수 어려움**.
    
- 예: 할인율 계산식을 바꾸려면 **모든 쿼리를 수정**해야 했음.
    

---

### ✅ 뷰에서 일원화한 뒤에는

- **모든 계산 로직을 `v_products_effective` 안에만 작성**.
    
- 화면/서비스 쪽에서는 `SELECT v.sale_price, v.final_price, v.dis_rate, v.dis_price …` 만 호출.
    
- 정책이 바뀌어도 뷰만 수정하면 전체가 동시에 반영됨.
    

즉, **Single Source of Truth (단일 진실 원본)** 구조가 되는 겁니다.

---

### 📐 예시

상품 리스트, 장바구니, 주문서 등 **모든 쿼리**가 이렇게 단순화:

```sql
`SELECT v.product_id, v.product_name, v.final_price, v.dis_rate, v.dis_price FROM v_products_effective v WHERE v.category_id = 100;`
```


---

👉 그래서 제가 말씀드린 “일원화”는

- **할인가격 계산**
    
- **fallback 처리 (sale=0 → buy_price)**
    
- **노출/재고/상태 필터링**
    

이런 로직을 전부 **뷰 한 곳에 모아 관리**한다는 뜻이에요.