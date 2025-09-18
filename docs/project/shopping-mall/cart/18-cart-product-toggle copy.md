---
layout: default
title: "[cart] 장바구니 선택삭제"
parent: cart
nav_order: 19
---


#
# 선택상품 삭제 시 ajax <-> 서버 파싱

# 1) 전송 규약(확정)

* 서버가 문자열만 받도록 했으니, **루트로 문자열만** 보내세요.

- **프런트:** `payload` 없이 **루트 키**로 보냄
```js
API("removeMany", { cart_ids: JSON.stringify(uniq_cart) });

```

- - `uniq_cart`는 배열 → **한 번만** `JSON.stringify` 하기.
        
- **서버:** 루트 `cart_ids`를 문자열(JSON)로 받고 `json_decode`
```php
$idsJson = (string)get_param('cart_ids', '[]');  // 최상위 키로 바로 수신
$ids = json_decode($idsJson, true) ?: []; // ['12','34'] 형태로 복원

// 정규화(문자열화 + 공백/빈값/중복 제거)
$ids = array_values(array_unique(array_filter(
    array_map(fn($v) => trim((string)$v), $ids),
    fn($v) => $v !== ''
)));

```
# 점검 팁

- **빈값이면**: 프런트에서 정말 루트로 보냈는지 확인하세요. (`payload.cart_ids`로 보내면 서버가 못 읽음)
    
- 네트워크 탭에서 **Request Payload / Form Data** 확인:
    
    - `cart_ids: ["12","34"]`가 보여야 정상.
    - 
# 2) 프런트 예시
```js
const cart_ids = [];
$(ROWS).each(function(){
  const $row = $(this);
  if ($row.find(CHK).prop("checked")) {
    const cart_id = $row.data("cart-id");
    if (cart_id) cart_ids.push(String(cart_id));
  }
});
if (!cart_ids.length) return;

const uniq_cart = Array.from(new Set(cart_ids));
if (!confirm(`선택한 ${uniq_cart.length}개 상품을 삭제하시겠습니까?`)) return;

$btn.prop("disabled", true);
API("removeMany", { cart_ids: JSON.stringify(uniq_cart) })
  .then(res => {
    if (!res || res.ok !== true) {
      alert(res && res.msg ? res.msg : "선택삭제 실패");
      return;
    }
    // 성공 처리...
  })
  .catch(() => alert("요청 실패"))
  .finally(() => $btn.prop("disabled", false));

```

# 3) 서버(핵심만)
```php
case 'removeMany':
    ensure_post();

    $idsJson = (string)get_param('cart_ids', '[]');
    $ids = json_decode($idsJson, true) ?: [];

    $ids = array_values(array_unique(array_filter(
        array_map(fn($v) => trim((string)$v), $ids),
        fn($v) => $v !== ''
    )));
    if (!$ids) { json_out(['ok'=>true, 'deleted'=>0]); break; }

    $sql = "CALL remove_shop_cart(:uid, :cid, @rc, @msg)";
    $st  = $dbh->prepare($sql);

    $ok = 0; $fail = [];
    foreach ($ids as $cid) {
        try {
            $st->execute([':uid'=>$user_id, ':cid'=>$cid]);
            while ($st->nextRowset()) {}
            $st->closeCursor();

            $out = $dbh->query("SELECT @rc AS rc, @msg AS msg")->fetch(PDO::FETCH_ASSOC) ?: [];
            $rc  = (int)($out['rc'] ?? -1);
            $msg =        ($out['msg'] ?? null);

            if ($rc === 0) $ok++; else $fail[] = ['cid'=>$cid, 'rc'=>$rc, 'msg'=>$msg];
        } catch (Throwable $e) {
            $fail[] = ['cid'=>$cid, 'rc'=>-999, 'msg'=>$e->getMessage()];
            try { while ($st->nextRowset()) {} $st->closeCursor(); } catch(Throwable $e2){}
        }
    }

    if (!$fail) json_out(['ok'=>true, 'deleted'=>$ok]);
    else json_out(['ok'=>false, 'code'=>100, 'msg'=>'일부 삭제 실패', 'deleted'=>$ok, 'fails'=>$fail], 200);
    break;

``` 

# 4) 빠른 디버깅 팁

- 서버에서 즉시 확인:
```php
json_out(['ok'=>false, 'msg'=>$idsJson ?: '(empty)']); break;

```

- 브라우저 **Network → Request Payload/Form Data**에서 `cart_ids` 값이 실제로 보이는지 체크.
    

# 5) 흔한 실수 체크리스트

- `payload.cart_ids`로 보내지 않기(서버는 루트만 읽음).
    
- `JSON.stringify` **두 번** 하지 않기.
    
- 변수명 혼용 금지: `$idsJson` vs `$ids_json`.
    
- 배열을 강제로 `(string)` 캐스팅하지 않기(“Array”가 되어 파싱 실패).