---
layout: default
title: "[cart] 장바구니 담기 수량 UPDATE"
parent: cart
nav_order: 17
---



#### $(document).trigger("cart:updated"); 는 jQuery의 **커스텀 이벤트 발생 코드**입니다.


`detail.php`에서 “장바구니 담기” 성공 후 **헤더 카운트만 즉시 갱신**하려면, 아래처럼 **이벤트 트리거(cart:updated)**를 그대로 활용하면 됩니다. 핵심은 **추가 API 응답에 cart_count를 포함**해 주고, 없을 때는 **헤더가 직접 카운트를 조회**하는 두 가지 경로를 다 여는 거예요.

### 1) detail.php (장바구니 담기 → 이벤트 트리거)
```html
<!-- detail.php (예시) -->
<button id="btnAddToCart">장바구니 담기</button>

<script>
// jQuery 사용 예시
$('#btnAddToCart').on('click', function () {
  const payload = {
    action: 'add',                 // 서버 API 설계에 맞게
    product_id: $('#product_id').val(),
    quantity: Number($('#qty').val() || 1),
    // 옵션/추가구성 등이 있으면 함께 전송
    // option_json: JSON.stringify(...),
  };

  $.post('/api/cart_proc.php', payload)
    .done(function (resp) {
      // resp 예시: { ok: true, cart_count: 5 }
      if (resp && resp.ok) {
        // (A) 응답에 cart_count 있으면 같이 보내기
        $(document).trigger('cart:updated', [resp.cart_count]);
        // UX 피드백
        toast('장바구니에 담았습니다.');
      } else {
        alert(resp?.msg || '담기에 실패했습니다.');
      }
    })
    .fail(function () {
      alert('네트워크 오류가 발생했습니다.');
    });
});
</script>

```

만약 jQuery가 아니라 `fetch`를 쓰신다면, `then`에서 동일하게 `document.dispatchEvent(new CustomEvent('cart:updated', { detail: { count } }))` 형태로 보내면 됩니다.

# 2) header.php (이벤트 리스너로 뱃지 갱신)

헤더는 두 가지를 처리하면 안정적입니다.

- **케이스 A:** `detail.php`가 새 카운트를 **넘겨준 경우** → 그대로 반영
    
- **케이스 B:** 새 카운트가 **없으면** → **API로 카운트 재조회** 후 반영
```html
<!-- header.php (예시) -->
<span id="cartCountBadge" class="badge">0</span>

<script>
// 카운트 표시 유틸
function renderCartCount(n) {
  const $badge = $('#cartCountBadge');
  $badge.text(n);
  $badge.toggleClass('hidden', !n);
}

// 서버에서 카운트만 가져오는 경량 API 권장 (예: action=count)
function fetchCartCount() {
  return $.get('/api/cart_proc.php', { action: 'count' })
    .then(function (resp) {
      // resp 예시: { ok: true, cart_count: 5 }
      if (resp && resp.ok) return resp.cart_count ?? 0;
      return 0;
    })
    .catch(function () { return 0; });
}

// 1) 페이지 초기 진입 시 1회 동기화(선택)
$(function () {
  fetchCartCount().then(renderCartCount);
});

// 2) 장바구니 변경 이벤트를 어디서든 수신
$(document).on('cart:updated', function (_evt, newCount) {
  if (typeof newCount === 'number') {
    // 케이스 A: detail.php가 넘겨준 카운트 사용
    renderCartCount(newCount);
  } else {
    // 케이스 B: 직접 조회
    fetchCartCount().then(renderCartCount);
  }
});
</script>

```

# 3) 구현 팁

- **응답에 cart_count 넣기 권장**: 장바구니 추가/삭제/수정 API에서 매번 최신 카운트를 내려주면 **추가 조회 없이 즉시 반영** 가능.
    
- **디바운스(선택)**: 사용자가 빠르게 여러 번 담을 때 이벤트를 너무 자주 처리하지 않도록 200~400ms 디바운스.
    
- **미니카트 동기화(선택)**: 헤더 배지뿐 아니라 미니카트 목록을 쓰신다면 `cart:updated`에서 목록도 부분 갱신.
    
- **에러 시 원복**: 낙관적 UI를 쓰면, 실패 시 이전 카운트로 되돌리는 처리(또는 재조회) 포함.



구체적으로:

1. **이벤트 트리거링**
    
    - `document` 객체를 대상으로 `"cart:updated"`라는 이름의 이벤트를 발생시킵니다.
        
    - 이 시점에서 실제로 아무 동작도 하지 않고, 단순히 **"지금 cart:updated라는 이벤트가 발생했다"**라는 신호를 던진 것.
        
2. **리스너와 연결**
    
    - 다른 코드에서 `$(document).on("cart:updated", function (event, data) { ... })` 같은 리스너를 등록해두면, 트리거된 순간 해당 핸들러가 실행됩니다.
        
    - 예: `header.php` 같은 곳에서 이 이벤트를 듣고 장바구니 카운트 뱃지를 새로 그려줌.
        
3. **데이터 전달도 가능**
    
    - `$(document).trigger("cart:updated", [5]);` 처럼 두 번째 인자로 배열/값을 넣으면 리스너 쪽에서 `function (event, newCount) { ... }` 형태로 받을 수 있습니다.
        
    - 즉 “장바구니가 갱신됐다 → 현재 카운트는 5개”라는 식의 정보까지 함께 전달 가능.
        
4. **장점**
    
    - **결합도 낮음**: detail.php, basket.php, header.php 같은 파일이 서로 직접 호출하지 않고도, “이벤트”라는 매개체로 통신 가능.
        
    - **여러 구독자 지원**: header 뱃지, 미니카트 팝업, 토스트 메시지 등 여러 모듈이 동시에 cart:updated를 구독해 반응 가능.
        

---

👉 정리하면,  
`$(document).trigger("cart:updated");` 는 **“장바구니 상태가 변경되었으니 관심 있는 모듈은 알아서 반응해라”**라는 신호를 broadcast하는 코드입니다.