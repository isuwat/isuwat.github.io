---
layout: default
title: "[option] 옵션 핵심 코드 스케치
parent: shop
nav_order: 30
---

# 옵션 핵심 코드 스케치


# 옵션 핵심 코드 스케치(요지)

### detail.php (옵션/히든필드/버튼)

```php
// 목록: CALL sp_get_product_variants(:pid)
<select id="product_option_id1" name="option1" required>
  <option value="*">- [필수] 옵션을 선택해 주세요 -</option>
  <option value="**" disabled>-------------------</option>
  <optgroup label="옵션">
    <?php foreach ($variants as $v):
      $disabled = (!$v['is_active'] || (int)$v['stock_qty']<=0) ? 'disabled' : '';
      $label = $v['option_label'] . ($disabled ? ' (품절)' : '');
    ?>
      <option value="<?= htmlspecialchars($v['sku_code']) ?>" <?= $disabled ?>>
        <?= htmlspecialchars($label) ?>
      </option>
    <?php endforeach; ?>
  </optgroup>
</select>
<input type="hidden" id="variant_id" name="variant_id" value="">
<button type="submit" class="js-buy" disabled>장바구니 담기</button>

```

### JS (선택→AJAX→상태반영)

```js
const sel = document.getElementById('product_option_id1');
const variantIdEl = document.getElementById('variant_id');
const priceEl = document.querySelector('.js-price');
const stockEl = document.querySelector('.js-stock');
const buyBtn = document.querySelector('.js-buy');

sel.addEventListener('change', async () => {
  const sku = sel.value;
  if (sku === '*' || sku === '**') { variantIdEl.value=''; buyBtn.setAttribute('disabled','disabled'); return; }
  try {
    const r = await fetch(`<?= url_to('api/cart_proc.php') ?>?action=variant_by_sku&sku=${encodeURIComponent(sku)}`);
    if (!r.ok) throw 0;
    const v = await r.json();
    variantIdEl.value = v.variant_id;
    priceEl && (priceEl.textContent = Number(v.price).toLocaleString());
    stockEl && (stockEl.textContent = v.stock_qty);
    (v.stock_qty>0) ? buyBtn.removeAttribute('disabled') : buyBtn.setAttribute('disabled','disabled');
  } catch { variantIdEl.value=''; buyBtn.setAttribute('disabled','disabled'); }
});

```


### cart_proc.php (API 요지)

```php
// GET variant_by_sku
$stmt = $dbh->prepare("CALL sp_get_variant_by_sku(:sku)");
// POST add (OUT 파라미터 rc/msg)
$call = $dbh->prepare("CALL sp_cart_add(:uid,:pid,:vid,:qty,@rc,@msg)");

```


추후 애드온을 붙일 때는 **`action=add_with_addons`**를 별도로 추가하고, 현재 구조(메인 `variant_id` 결정 → 폼 전송)에 **addons_json**만 더해주면 됩니다. 기존 코드는 그대로 재사용.

## 체크리스트(메인만)

-  옵션 미선택 시 구매 버튼 비활성 + 경고
    
-  품절/비활성 옵션은 선택 불가(비활성) + “(품절)” 표기
    
-  선택 시 가격/재고/이미지 즉시 갱신
    
-  장바구니 저장 시 스냅샷(`variant_id, sku, unit_price, option_text`)
    
-  주문 성공시에만 재고 차감, 실패 롤백