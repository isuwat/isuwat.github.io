---
layout: default
title: "[css] 이미지 대체"
parent: Project
nav_order: 7
---


# 대체 이미지 표시

## CSS로 대체 (이미지 없는 경우 background)

```html
<div class="thumbnail <?= empty($row['product_img']) ? 'no-img' : '' ?>">     
<?php if (!empty($row['product_img'])): ?>         
<img src="<?= htmlspecialchars($row['product_img']) ?>" alt="상품 이미지">     
<?php endif; ?> 
</div>
```

```css
.thumbnail.no-img {     
	width:71px; height:71px;     
	background:url('/images/no_image.png') center center no-repeat;     
	background-size:cover; 
}
```

``
- 상품 이미지가 없으면 `<img>` 자체를 출력하지 않고, CSS 배경으로 처리    
- HTML이 더 깔끔해지고, 성능상 이미지 요청도 줄어듦
