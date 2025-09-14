---
layout: default
title: "[order] 결제금액"
parent: order
nav_order: 1
---

# 쇼핑몰 상품 결제 금액 정보

Here are payment developmnet tips.

✅ 왜 orders 테이블에 저장해야 하나요?

order_items는 상품별 데이터만 저장합니다.
반면, 주문 전체에 대한 요약 정보는 orders 테이블이 담당해야 합니다.

이런 정보들은:

* 결제 시점에 계산이 완료되고,
* 추후 주문 상태가 바뀌어도 변하지 않아야 하며
* 빠른 조회를 위해서도 별도로 저장해두는 것이 좋습니다.

---

✅ orders 테이블에 추가할 필드 제안

```sql
ALTER TABLE orders
ADD COLUMN product_total INT NOT NULL COMMENT '상품 합계 금액',
ADD COLUMN shipping_fee INT NOT NULL COMMENT '배송비',
ADD COLUMN total_payment INT NOT NULL COMMENT '최종 결제 금액 (상품합계 + 배송비)';
```

✅ 저장 시점

이 금액들은 주문을 저장하는 순간, 또는 결제 직전 다음처럼 계산되어야 합니다:

```php
$product_total = array_sum(array_map(fn($item) => $item['quantity'] * $item['unit_price'], $order_items));
$shipping_fee = ($product_total >= 50000) ? 0 : 3000;
$total_payment = $product_total + $shipping_fee;
```

✅ UI에 표시 예시

```html
<table class="table table-bordered">
  <tr><th>상품 합계</th><td><?= number_format($order['product_total']) ?> 원</td></tr>
  <tr><th>배송비</th><td><?= number_format($order['shipping_fee']) ?> 원</td></tr>
  <tr class="table-light fw-bold"><th>최종 결제금액</th><td><?= number_format($order['total_payment']) ?> 원</td></tr>
</table>
```
