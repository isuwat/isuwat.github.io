---
layout: default
title: "[cart] 바로구매-전체구매-선택구매-옵션 공통로직"
parent: shop
nav_order: 23
---



# 주문-바로구매-전체구매-옵션

맞아요—**옵션이 얽히는 바로구매**는 “카트에 한 줄 생성 → 그 줄의 `cart_id`(PK) 리턴 → 주문서에 `selected_cart_ids[]`로 전달”이 가장 안전하고 확장성 있는 구조예요.  
옵션 조합/추가구성/가격·재고 검증 모두를 **장바구니 공통 로직**으로 통일할 수 있습니다.

아래처럼 잡으면 깔끔합니다.

# 백엔드 권장 설계

## 1) 카트 row의 유니크 기준

- `UNIQUE (user_id, product_id, option_hash)`
    
    - `option_hash` = 옵션 조합/추가구성/커스텀 입력까지 포함한 **표준화 문자열의 해시**
        
    - 같은 상품이라도 옵션이 다르면 **별도 row** 유지, 같으면 **수량 가산**
        

## 2) add API (cart_api.php?action=add)

- **입력**: `product_id`, `qty`, `option_payload`(JSON)
    
    - 서버에서 `option_payload` → `option_hash` 생성, 가격/재고 검증
        
- **동작**:
    
    - (없음) INSERT → `quantity = qty`
        
    - (있음) UPDATE → `quantity = quantity + qty`
        
- **응답**:

```js
{
  "ok": true,
  "cart_id": 12345,            // 생성/갱신된 row
  "quantity_after": 4,         // 증가 후 최종 수량 (표시용)
  "option_hash": "abc123"      // (디버깅/로깅용)
}

```
    
- 가능하면 프로시저 OUT 파라미터로 `cart_id`, `quantity_after`를 바로 돌려주세요.  
    (어려우면 add 후 SELECT로 조회)
    

## 3) 주문서 진입

- 프런트는 add 성공 시:
    
    - `selected_cart_ids[] = cart_id` 만 POST
        
    - 주문서는 **shop_cart에서 해당 row**를 읽어 **옵션·수량·가격** 재계산
        

# 프런트(예시)

## detail.php – 바로구매

```js
function set_direct_buy() {
  const params = new URLSearchParams(location.search);
  const product_id = params.get('product_id');
  const qty = Math.max(1, parseInt(document.getElementById('quantity')?.value || '1', 10));

  // 옵션 수집 예시 (사이트 구조에 맞게 조정)
  const optionPayload = collectOptions(); // { color:'red', size:'M', addOns:[...], text: ... }

  $.ajax({
    url: '../api/cart_api.php?action=add',
    method: 'POST',
    dataType: 'json',
    data: {
      product_id,
      qty,
      option_payload: JSON.stringify(optionPayload)
    },
    success: function(res){
      if (!res?.ok) { alert('담기 실패'); return; }
      const cart_id = res.cart_id;

      // 해당 cart_id만 주문서로
      const f = document.createElement('form');
      f.method = 'post';
      f.action = '../order/order_form.php';
      const mk=(n,v)=>{const i=document.createElement('input');i.type='hidden';i.name=n;i.value=v;f.appendChild(i);};
      mk('mode','direct_buy');
      mk('scope','selected');
      mk('selected_cart_ids[]', String(cart_id));
      document.body.appendChild(f);
      f.submit();
    },
    error: () => alert('네트워크 오류')
  });
}

```

## basket.php – 전체/선택 주문

- **가능하면 `cart_id` 기반**으로 넘기세요(옵션 중복/세트 구분이 정확).
    
- 이미 렌더링된 각 행에 `data-cart-id`를 심어두고 수집:
    

```js
function collectCartIds(all=true){
  const rows = document.querySelectorAll('#cart-rows .ec-base-prdInfo.gCheck');
  const ids = [];
  rows.forEach(row => {
    const checked = row.querySelector('input[name="basket_product_normal_type_normal"]').checked;
    if (all || checked) {
      const cid = row.getAttribute('data-cart-id'); // 렌더링 시 주입
      if (cid) ids.push(cid);
    }
  });
  return Array.from(new Set(ids));
}

$('#btn-order-all, #btn-order-all-bottom').on('click', function(e){
  e.preventDefault();
  const ids = collectCartIds(true);
  if (!ids.length) return;

  const f = document.createElement('form');
  f.method='post'; f.action='./order_form.php';
  const mk=(n,v)=>{const i=document.createElement('input');i.type='hidden';i.name=n;i.value=v;f.appendChild(i);};
  mk('mode','cart_all');
  ids.forEach(id => mk('selected_cart_ids[]', id));
  document.body.appendChild(f); f.submit();
});

$('#btn-order-selected, #btn-order-selected-bottom').on('click', function(e){
  e.preventDefault();
  const ids = collectCartIds(false);
  if (!ids.length) { alert('선택된 상품이 없습니다.'); return; }

  const f = document.createElement('form');
  f.method='post'; f.action='./order_form.php';
  const mk=(n,v)=>{const i=document.createElement('input');i.type='hidden';i.name=n;i.value=v;f.appendChild(i);};
  mk('mode','cart_selected');
  ids.forEach(id => mk('selected_cart_ids[]', id));
  document.body.appendChild(f); f.submit();
});

```

# 주문서(order_form.php) 처리 요약

- `selected_cart_ids[]` 수신 → `shop_cart`에서 해당 row들을 조회
    
- 각 row의 `product_id`, `option_hash`로 **shop_product/옵션 테이블 재조회**
    
- **가격/재고/최소·최대구매수량/프로모션** 재검증 → 실패 시 사용자에게 조정 안내
    
- 배송비/쿠폰/적립금 로직은 기존 장바구니 로직 그대로 재사용
    

# 체크리스트

-  옵션 조합을 **option_hash**로 표준화 (프론트 payload → 서버 검증)
    
-  `add` 응답에 `cart_id`, `quantity_after` 포함
    
-  카트 행 렌더링에 `data-cart-id` 포함(선택/전체 주문 시 활용)
    
-  주문서에서 **반드시 서버 기준**으로 가격·재고·정책 재계산
    
-  동일 상품 중복 row 허용(옵션/추가구성 구분) + 같은 옵션은 가산
    

이렇게 가시면 **바로구매(옵션 포함)와 장바구니 주문 플로우를 하나의 질서로** 안정적으로 운영할 수 있습니다.