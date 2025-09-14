---
layout: default
title: "[cart] 바로구매 장바구니 구매 로직"
parent: cart
nav_order: 12
---


# 바로구매 장바구니 구매 로직

## 백엔드 권장 설계

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
```json
	{   "ok": true,   
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

ver2
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
      const mk=(n,v)=>{const i=document.createElement('input');
      i.type='hidden';i.name=n;i.value=v;f.appendChild(i);};
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
``
ver1
```js
<script>
// detail.php - "바로구매" = 카트에 담고, 해당 상품만 주문서로 이동
function set_direct_buy() {
  // 1) 상품/옵션/수량 수집
  const params = new URLSearchParams(location.search);
  const product_id = params.get('product_id');         // 쿼리로 넘기는 구조일 때
  const qty = parseInt((document.getElementById('quantity')?.value || '1'), 10);

  if (!product_id || isNaN(qty) || qty < 1) {
    alert('상품 정보가 올바르지 않습니다.');
    return;
  }

  // 옵션 값 예시(사이트 구조에 맞춰 name/select 등을 조정하세요)
  // ex) <select name="option_color">, <select name="option_size"> 등
  const optionInputs = document.querySelectorAll('[name^="option_"]');
  const options = {};
  optionInputs.forEach(el => { options[el.name] = el.value; });

  // 2) 카트에 담기 (cart_proc.php가 사용하는 형식에 맞추어 전송)
  //    - cart_id 반환 시: 해당 cart_id만 주문서로 넘김 (권장)
  //    - cart_id가 없다면: 폴백으로 주문서에 direct payload 전송
  const payload = {
    action: 'add',
    product_id,
    qty,
    options: JSON.stringify(options || {}),
    want_cart_id: 1              // 서버가 가능하면 cart_id를 함께 반환해 달라는 힌트
  };

  // 로딩 방지 중복 클릭 억제
  const btn = document.getElementById('btnBuyNow') || document.querySelector('.btnSubmit.buy-now');
  if (btn) btn.disabled = true;

  $.ajax({
    url: '../api/cart_proc.php',
    method: 'POST',
    dataType: 'json',
    data: payload,
    success: function(res) {
      // 3-A) 선호 경로: cart_id가 오면 그 항목만 주문서로 이동
      if (res && (res.cart_id || (res.data && res.data.cart_id))) {
        const cart_id = res.cart_id || res.data.cart_id;

        // 주문서로 "선택항목만" 넘기기 (selected_cart_ids[]=id 형태 예시)
        const f = document.createElement('form');
        f.method = 'post';
        f.action = '../order/order_form.php';

        const mk = (n, v) => {
          const i = document.createElement('input');
          i.type = 'hidden'; i.name = n; i.value = v;
          f.appendChild(i);
        };

        mk('mode', 'direct_buy');              // 직결제 모드 의도
        mk('scope', 'selected');               // 선택항목만
        mk('selected_cart_ids[]', String(cart_id));

        document.body.appendChild(f);
        f.submit();
        return;
      }

      // 3-B) 폴백: cart_id가 없다면 주문서에 바로 payload 전달
      //  - 서버에서 items_json을 받아 카트/세션과 독립적으로 단건 결제 라우팅
      const items = [{
        product_id: product_id,
        qty: qty,
        options: options
      }];

      const f = document.createElement('form');
      f.method = 'post';
      f.action = '../order/order_form.php';

      const mk = (n, v) => {
        const i = document.createElement('input');
        i.type = 'hidden'; i.name = n; i.value = v;
        f.appendChild(i);
      };

      mk('mode', 'direct_buy');               // 직결제
      mk('scope', 'single');                  // 단일항목
      mk('items_json', JSON.stringify(items));// 주문서가 파싱하도록 전달

      document.body.appendChild(f);
      f.submit();
    },
    error: function(xhr) {
      alert('네트워크 오류 (' + xhr.status + ')');
    },
    complete: function() {
      if (btn) btn.disabled = false;
    }
  });
}
</script>

```


## basket.php – 전체/선택 주문

- **가능하면 `cart_id`  기반**으로 넘기세요(옵션 중복/세트 구분이 정확).
    
- 이미 렌더링된 각 행에 `data-cart-id`를 심어두고 수집:
    
```js
function collectCartIds(all=true){
  const rows = document.querySelectorAll('#cart-rows .ec-base-prdInfo.gCheck');
  const ids = [];
  rows.forEach(row => {const checked = row.querySelector('input[name="basket_product_normal_type_normal"]').checked;
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
	  const mk=(n,v)=>{
		  const i=document.createElement('input');
		  i.type='hidden';
		  i.name=n;
		  i.value=v;
		  f.appendChild(i);
	  };
  mk('mode','cart_all');
  ids.forEach(id => mk('selected_cart_ids[]', id));
  document.body.appendChild(f); f.submit();
});

$('#btn-order-selected, #btn-order-selected-bottom').on('click', function(e){
  e.preventDefault();
  const ids = collectCartIds(false);
  if (!ids.length) { alert('선택된 상품이 없습니다.'); return; }

  const f = document.createElement('form');
  f.method='post'; 
  f.action='./order_form.php';
  const mk=(n,v)=>{
	  const i=document.createElement('input');
	  i.type='hidden';
	  i.name=n;
	  i.value=v;
	  f.appendChild(i);
  };
  mk('mode','cart_selected');
  ids.forEach(id => mk('selected_cart_ids[]', id));
  document.body.appendChild(f); 
  f.submit();
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