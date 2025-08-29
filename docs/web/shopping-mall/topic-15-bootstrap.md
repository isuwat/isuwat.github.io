---
layout: default
title: "[common] 공통 부트스트랩 정의"
parent: shop
grand_parent: Web
nav_order: 15
---

## 이 문서는 공통 부트스트랩 파일과 페이지 include 방식을 설명하기 위해 작성된 글입니다.

### 1. 공통 부트스트랩

- `BASE_PATH = dirname(__DIR__)`: 어느 파일에서 include해도 **프로젝트 루트**를 확정.
- `path_to() / url_to()`: 파일시스템 경로/웹 URL을 항상 루트 기준으로 생성.
- 공통 라이브러리(`libPdo.php` 등)는 여기에 모아서 **중복 로드 방지**.

```php
<?php
require_once __DIR__ . '/../common/bootstrap.php';
?>
```

### __DIR__는 PHP의 매직 상수(magic constant) 중 하나입니다.
--- 
### 📌 의미
* __DIR__ 는 현재 실행 중인 파일이 위치한 디렉터리 경로(절대경로)를 반환합니다.
* 즉, 그 PHP 파일이 어디서 include/require 되든 상관없이, 그 파일 자체가 있는 폴더를 기준으로 경로를 잡아줍니다.

### 📌 예시
```css
구조
/var/www/html/
 ├─ common/
 │   └─ bootstrap.php
 ├─ product/
 │   └─ list.php
 └─ order/
     └─ basket.php
```

### basket.php 안에서
```php
require_once __DIR__ . '/../common/bootstrap.php';
```
* __DIR__ = /var/www/html/order

* 따라서 실제 include 되는 경로 =  
  /var/www/html/order/../common/bootstrap.php → /var/www/html/common/bootstrap.php

  이렇게 하면 basket.php를 /order에서 실행하든, /product/list.php에서 include하든 항상 올바른 상대 경로가 계산됩니다.

### 📌 __DIR__의 장점

1. 상대경로 오류 방지

* 단순히 require '../common/bootstrap.php';라고 쓰면 현재 실행 위치에 따라 깨질 수 있음.

* __DIR__를 쓰면 파일의 실제 물리적 위치를 기준으로 경로가 잡히므로 안전.

2. 유지보수 편리

* 프로젝트 이동, 디렉터리 이름 변경에도 잘 동작.

### ✅ 결론
* __DIR__ = “이 PHP 파일이 있는 디렉터리 경로”  
  그래서 require_once __DIR__ . '/../common/bootstrap.php';는  
  현재 파일의 디렉터리에서 한 단계 위로 올라가서 common/bootstrap.php를 불러라는 의미입니다.



### 2. 각 페이지에서 공통 파일 불러오기

모든 페이지 최상단에서 부트스트랩을 require.

- **product/list.php, product/detail.php**

```php
require_once __DIR__ . '/../common/bootstrap.php';
```

- **order/basket.php**

```php
require_once __DIR__ . '/../common/bootstrap.php';
```

### 3. 왜 같은 한 줄인가요?

- `__DIR__`는 "이 파일이 있는 폴더".
- `..`는 "그 상위 폴더".
- 따라서 product/든 order/든 한 단계 위로 올라가면 공통 디렉토리 `common/`에 접근할 수 있습니다.
- 항상 **정확한 경로**로 include 보장.


### 공통 부트스트랩 만들기&#x20;

`common/bootstrap.php`

```
<?php
// 앱이 도메인 루트가 아니라면 '/shop' 처럼 설정
if (!defined('APP_BASE')) {
  define('APP_BASE', '/'); // 예: '/shop'
}

// 이 파일의 상위(=프로젝트 루트) 절대경로
if (!defined('BASE_PATH')) {
  define('BASE_PATH', dirname(__DIR__)); // /var/www/html 같은 루트
}

// ----- 경로/URL 헬퍼 -----
if (!function_exists('path_to')) {
  // 프로젝트 루트(BASE_PATH) 기준 파일 시스템 경로
  function path_to(string $rel): string {
    return rtrim(BASE_PATH, DIRECTORY_SEPARATOR) . DIRECTORY_SEPARATOR . ltrim($rel, DIRECTORY_SEPARATOR);
  }
}

if (!function_exists('url_to')) {
  // 앱 베이스(APP_BASE) 기준 URL
  function url_to(string $path): string {
    return rtrim(APP_BASE, '/') . '/' . ltrim($path, '/');
  }
}

// ----- 공통 의존성 로드 (여기서 한 번만) -----
require_once path_to('lib/libPdo.php');
// 필요시: require_once path_to('lib/session.php');
// 필요시: require_once path_to('config/env.php');

```

### 4. header.php의 카테고리 링크를 절대 경로로

* 상대경로 list.php?... 대신 헬퍼로 절대 URL을 생성하게 바꿉니다.

```html
<nav id="categorymenu_shop" class="categorymenu menuelem">
  <h2 class="mainTitle">SHOP</h2>
  <ul class="-flex">
    <li class="-d1 d1ddm"><a href="<?php echo url_to('product/list.php'); ?>?cate_no=680">브랜드</a></li>
    <li class="-d1 d1ddm"><a href="<?php echo url_to('product/list.php'); ?>?cate_no=44">여성토이</a></li>
    <li class="-d1 d1ddm"><a href="<?php echo url_to('product/list.php'); ?>?cate_no=46">남성토이</a></li>
    <!-- … 나머지도 동일 패턴 … -->
  </ul>
</nav>
```

> 이렇게 하면 product/에서도 order/에서도 항상 /product/list.php?...로 이동합니다.<br>
> (APP_BASE가 /shop이면 /shop/product/list.php?...가 됩니다.)

