---
layout: default
title: "[cart] 장바구니 상품목록 토글"
parent: cart
nav_order: 18
---



# 장바구니 상품목록 토글

안보이도록 처리 display: inline-block; -> display: none;

```css
/* 장바구니 상품 접힘/닫힘 토글 */

.ec-base-fold.theme1 > .title:after {

    content: "";

    position: absolute;

    top: 20px;

    right: 5px;

    overflow: hidden;

    /* display: inline-block; */

    display: none;

    width: 10px;

    height: 10px;

    border-left: 1px solid #000;

    border-bottom: 1px solid #000;

    -webkit-transform: rotate(-45deg);

    transform: rotate(-45deg);

}
```

