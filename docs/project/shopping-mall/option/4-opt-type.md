---
layout: default
title: "[option] 옵션 유형"
parent: option
nav_order: 4
---

# 옵션 유형

* 기본: 셀렉트박스 선택
* 추가 애드온: 추가구성상품 셀렉트 박스

### 카페24 상품 옵션 UI는 크게 두 가지 방식으로 나뉘어요:

1. **메인 기본형 옵션 셀렉트박스**
    
    - 상품 자체의 구매 필수 옵션 (예: 색상, 사이즈)
        
    - `<select>`로 렌더링되고, `- [필수] 옵션을 선택해 주세요 -` 같은 안내 문구 포함
        
    - 선택 결과가 `variant_id`(SKU) 매칭으로 이어져 **가격/재고/이미지**까지 반영
```html
<select name="option1" id="product_option_id1" required>
  <option value="*">- [필수] 옵션을 선택해 주세요 -</option>
  <option value="P000ABC000B">블루 (M)</option>
  <option value="P000ABC000C">블루 (L)</option>
</select>

```

2. **추가구성상품(애드온) 셀렉트박스**

- 본상품과 함께 구매할 수 있는 **부가 상품**
    
- “추가구성상품” 블록 안에 `<select>`로 노출되며, 기본값은 `- [선택] 상품 선택 -`
    
- 선택 시 장바구니/주문에서는 **메인과 별도의 라인아이템**으로 처리되지만, 화면에서는 묶음으로 보여줌
```html
<select name="addproduct_option_name_1785" id="addproduct_option_id_1785_1">
  <option value="*" selected>- [선택] 상품 선택 -</option>
  <option value="P0000CQQ000C">추가 파우더 30g (+4,000원)</option>
</select>

```

📌 정리하면:

- **필수 구매 옵션 = 메인 옵션 셀렉트**
    
- **선택 구매 옵션 = 추가구성(애드온) 셀렉트**
    

👉 그래서 카페24 쇼핑몰의 옵션 UI 구조는 **“메인 기본형 셀렉트박스 + (선택적) 애드온 셀렉트박스”** 두 가지 축으로 되어 있어요.