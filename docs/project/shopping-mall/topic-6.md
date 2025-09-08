---
layout: default
title: "[db] 제약 조건"
parent: shop
nav_order: 6
---

# 장바구니 상품 개수 제약 조건

2) 체크 제약(>0) 추가

테이블 제약으로 추가하는 게 가장 호환성이 좋습니다.
```sql
ALTER TABLE `kpp_oncash`.`shop_cart`
  ADD CONSTRAINT `chk_shop_cart_quantity_positive` CHECK (`quantity` > 0);
  ```