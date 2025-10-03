---
layout: default
title: "[JS] htmlspecialchars()"
parent: JS
nav_order: 5
---

# htmlspecialchars()



**HTML 이스케이프 헬퍼**예요.  
`htmlspecialchars()`를 감싼 짧은 함수로, 화면에 찍을 문자열을 안전하게 바꿔 **XSS나 깨짐을 방지**합니다.

```php
function h($s){
  return htmlspecialchars((string)$s, ENT_QUOTES, 'UTF-8');
}
```

- `(string)$s` : 무엇이 와도 문자열로 변환    
- `ENT_QUOTES` : 작은따옴표(`'`)와 큰따옴표(`"`)까지 전부 이스케이프    
- `'UTF-8'` : 인코딩 지정(한글/이모지 깨짐 방지)    

### 무엇이 바뀌나?

- `&` → `&amp;`    
- `<` → `&lt;`    
- `>` → `&gt;`    
- `"` → `&quot;`    
- `'` → `&#039;` (ENT_QUOTES 덕분)    

### 언제 쓰나?

템플릿에서 **사용자/DB 입력값을 그대로 찍을 때 항상**:

```php
<img src="<?= h($product['product_thumb_img_url']) ?>"
     alt="<?= h($product['product_name']) ?>">

<div class="name"><?= h($product['product_name']) ?></div>
```

### 주의(컨텍스트별)

- **URL 파라미터**: 링크의 `id` 같은 값은 `rawurlencode()`로 인코딩한 뒤, 속성에 넣을 때는 전체를 `h()`로 감싸면 안전합니다.

```php
<a href="/product/detail.php?id=<?= h(rawurlencode($id)) ?>">상세</a>

```

- **JS에 값 주입**: HTML 이스케이프 대신 `json_encode($val)`로 넣는 게 안전합니다.    
- **이미 이스케이프된 문자열**을 다시 `h()`로 감싸면 **이중 이스케이프**가 되니 주의.    

요약: `h()`는 “템플릿에서 값 찍을 땐 무조건 감싸는” 안전 기본기라고 보시면 됩니다.