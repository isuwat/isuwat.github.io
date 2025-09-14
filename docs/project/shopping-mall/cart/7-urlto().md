---
layout: default
title: "[cart] 절대 경로 url_to()"
parent: cart
nav_order: 7
---

## 장바구니 카운트 갱신 구조

### 1. 문제 배경
기존에는 `basket.php`, `list.php`, `main.php` 등 위치마다 `depth`(`..` or `.`)를 계산해 `cart_proc.php`를 호출했다. 이 방식은 페이지 디렉토리 구조가 달라질 때 깨질 수 있었다.

### 2. 해결 방법: 공통 부트스트랩 적용
- `bootstrap.php`에 `url_to()` 헬퍼를 정의
- 어느 페이지에서든 `url_to('api/cart_proc.php')` 호출 시 `/api/cart_proc.php` 절대경로가 생성됨
- 더 이상 `depth` 계산 불필요

### 3. 장바구니 카운트 스크립트
```php
<script>
    const CART_API_BASE = "<?php echo url_to('api/cart_proc.php'); ?>";

    (function(){
        const $ = window.$JQ || jQuery;

        function refreshCartCount() {
            $.getJSON(CART_API_BASE, { action: "list" })
                .done(function(res){
                    if (res.ok) {
                        let totalQty = 0;
                        (res.items || []).forEach(it => totalQty += (it.quantity || 0));
                        $(".xans_myshop_main_basket_cnt").text(totalQty);
                    }
                })
                .fail(function(xhr){
                    console.warn("[cartCount] failed", xhr.responseText);
                });
        }

        $(document).ready(refreshCartCount);
    })();
</script>
```

### 4. 기대 효과
- list.php, basket.php, main.php 어디서 include 해도 동일하게 작동
- 디렉토리 구조 변경에도 안전
- 유지보수성 향상

