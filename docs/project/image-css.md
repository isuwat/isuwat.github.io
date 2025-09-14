---
layout: default
title: "[css] 이미지 대체"
parent: Project
nav_order: 3
---


# 대체 이미지 표시

## CSS로 대체 (이미지 없는 경우 background)

말씀하신 **CSS로 대체**하는 방식은 “이미지 값이 없으면 `<img>` 자체를 출력하지 않고, CSS 배경으로 placeholder를 보여주는” 구조예요.  
이 방식은 카페24 기본 더미 이미지를 쓰지 않고도 **깨짐 없는 안전한 썸네일 처리**가 가능합니다.

```html
<div class="thumbnail <?= empty($row['product_img']) ? 'no-img' : '' ?>">     
<?php if (!empty($row['product_img'])): ?>         
<img src="<?= htmlspecialchars($row['product_img']) ?>" alt="상품 이미지">     
<?php endif; ?> 
</div>
```

```css
thumbnail {
    width: 71px;
    height: 71px;
    display: inline-block;
    overflow: hidden;
    border: 1px solid #eee;
    border-radius: 4px;
}
```


```css
.thumbnail.no-img {     
	background:url('/images/no_image.png') center center no-repeat;     
	background-size:cover; 
}
```

``
- 상품 이미지가 없으면 `<img>` 자체를 출력하지 않고, CSS 배경으로 처리    
- HTML이 더 깔끔해지고, 성능상 이미지 요청도 줄어듦
