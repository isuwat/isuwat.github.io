---
layout: default
title: snippet
parent: Cheat Sheet
nav_order: 1
---

# 스니펫이란?

“스니펫(snippet)”은 **코드나 문서의 짧은 발췌본**을 의미합니다.

- 긴 코드 전체를 다 보여주기보다는, **핵심적인 부분만 잘라서 보여주는 샘플**이라고 생각하시면 됩니다.
    
- 문서나 Jira 티켓, 기술 블로그에서 주로 **예시 코드**나 **중요한 포인트만 담은 짧은 코드 조각**을 “스니펫”이라고 부릅니다.
    

---

### 예시

장바구니 전체 코드(수백 줄)는 복잡하지만, **핵심만 보여주는 스니펫**은 이렇게 됩니다:

```js
API("update", { cart_id, qty: val })
  .then(res => {
    const cid = String(res.cart_id);
    const qty_after = parseInt(res.qty_after, 10) || val;
    postToOrderForm([{ cart_id: cid, qty: qty_after }], {
      mode: 'cart_json_one',
      scope: 'direct'
    });
  });

```

여기서는 **“바로구매 버튼 클릭 → 수량 업데이트 → 주문서 이동”** 핵심 흐름만 담은 짧은 코드가 스니펫입니다.

---

👉 정리하면:  
**스니펫 = 긴 코드 중 핵심만 발췌한 짧은 코드 조각**
