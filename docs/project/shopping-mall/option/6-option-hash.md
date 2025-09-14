---
layout: default
title: "[option] 옵션 해시"
parent: option
nav_order: 6
---


# 옵션 해시

**option_hash 생성 규칙 + 구현 예시**를 “일관성 있게” 쓰는 게 핵심입니다. 아래처럼 잡아두면 실무에서 안전해요.

---

# 1) 규칙(권장)

1. 옵션 페이로드를 **Key-Value 평탄화**
    
2. **Key 사전순 정렬**
    
3. 값은 **트림/소문자/표준화**(예: 공백·전각 제거)
    
4. `"key=value"` 조각을 `'&'`로 연결 → **canonical string**
    
5. 해시: **SHA1(or MD5)**
    

예) `{ color:'Red ', size:'M', addOns:['gift','wrap'] }`  
→ 평탄화: `{ addOns.0:'gift', addOns.1:'wrap', color:'red', size:'m' }`  
→ 정렬/연결: `"addOns.0=gift&addOns.1=wrap&color=red&size=m"`  
→ `SHA1(...)` = `40자 hex` → **option_hash**

> 옵션이 전혀 없으면 빈 문자열 `''`(또는 `"__noopt__"`)로 통일.

---

# 2) 서버(PHP) 해시 함수 예시

```php
function make_option_hash($payload) {
  // $payload: 배열 or JSON 문자열
  if (is_string($payload)) {
    $data = json_decode($payload, true) ?: [];
  } else {
    $data = $payload ?: [];
  }

  // 평탄화
  $flat = [];
  $walk = function($k, $v) use (&$flat, &$walk) {
    if (is_array($v)) {
      $i = 0;
      foreach ($v as $kk => $vv) {
        $key = is_int($kk) ? "{$k}.{$i}" : "{$k}.{$kk}";
        $walk($key, $vv);
        $i++;
      }
    } else {
      // 표준화: 문자열이면 트림/소문자
      if (is_string($v)) {
        $v = mb_strtolower(trim($v), 'UTF-8');
        $v = preg_replace('/\s+/u',' ', $v); // 공백 정규화
      }
      $flat[$k] = (string)$v;
    }
  };

  foreach ($data as $k => $v) $walk($k, $v);

  if (empty($flat)) return ''; // 옵션 없음

  ksort($flat, SORT_STRING);
  $parts = [];
  foreach ($flat as $k => $v) $parts[] = "{$k}={$v}";
  $canonical = implode('&', $parts);

  return sha1($canonical); // or md5($canonical)
}

```
---

# 3) 프런트(detail.php)에서 보내기(선택)

- 프런트는 **그냥 원본 옵션 payload**만 전송하고,
    
- **서버에서 위 함수로 해시**를 만드는 것이 가장 안전합니다.
    
```js
// 예: detail.php
function collectOptions() {
  return {
    color: $('#optColor').val(),
    size:  $('#optSize').val(),
    addOns: $('.addon:checked').map((_,el)=>$(el).val()).get()
  };
}
$.post('../api/cart_api.php?action=add', {
  product_id,
  qty,
  option_payload: JSON.stringify(collectOptions())
});

```

---

# 4) MySQL에서 해시(보조적)

**MySQL 8+**라면 JSON 컬럼이 있을 때 서버에서 직접 해시를 만들 수도 있습니다(복잡도↑). 실무에선 **PHP에서 생성**을 권장합니다.

```sql
-- canonical string이 준비돼 있다면
UPDATE shop_cart
SET option_hash = SHA1(canonical_string);

```

> canonical_string을 MySQL에서 만들려면 JSON_KEYS/JSON_EXTRACT로 키를 뽑아 정렬→재조합이 필요해 유지보수가 어렵습니다. **애플리케이션 레이어에서 생성**이 현실적입니다.

---

# 5) 마이그레이션 순서(기존 테이블에 적용)

1. `option_hash` 추가 (NULL 허용)
    

```sql
ALTER TABLE shop_cart ADD COLUMN option_hash VARCHAR(64) NULL AFTER product_id;

```

2. 기존 row 채우기
    
    - 옵션 없는 row → `''`
        
    - 옵션 있는 row → 애플리케이션/배치 스크립트로 `make_option_hash()` 호출해 업데이트
        

```php
// 배치 스케치
$rows = $dbh->query("SELECT cart_id, option_payload FROM shop_cart")->fetchAll(PDO::FETCH_ASSOC);
foreach ($rows as $r) {
  $hash = make_option_hash($r['option_payload']); // 없으면 ''
  $st = $dbh->prepare("UPDATE shop_cart SET option_hash=:h WHERE cart_id=:id");
  $st->execute([':h'=>$hash, ':id'=>$r['cart_id']]);
}

```
3. `NOT NULL` + 기본값 설정
    
```sql
UPDATE shop_cart SET option_hash='' WHERE option_hash IS NULL;

ALTER TABLE shop_cart
  MODIFY option_hash VARCHAR(64) NOT NULL DEFAULT '';

```

4. **UNIQUE 인덱스** 추가
    

```sql
ALTER TABLE shop_cart
  ADD UNIQUE KEY uniq_user_product_option (user_id, product_id, option_hash);

```

---

# 6) INSERT/UPSERT 예시

```sql
-- PHP에서 option_hash 만든 뒤 사용
INSERT INTO shop_cart (user_id, product_id, option_hash, quantity)
VALUES (:uid, :pid, :h, :qty)
ON DUPLICATE KEY UPDATE
  quantity = quantity + VALUES(quantity),
  updated_at = CURRENT_TIMESTAMP;

```

---

# 7) 체크리스트

-  **서버에서** 해시 생성(프런트 값 신뢰 X)
    
-  옵션 없음 → `''` 통일
    
-  동일 옵션 정의를 어디서든 같은 방식으로 평탄화/정렬/표준화
    
-  UNIQUE(user_id, product_id, option_hash)로 가산 규칙 보장
    
-  바로구매도 `option_payload` 포함해 add → `cart_id` 반환 → 주문서 진입