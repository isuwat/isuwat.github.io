---
layout: default
title: PHP include
parent: PHP
nav_order: 4
---


# include

## 1. `__DIR__` 기반 `require_once`

`require_once __DIR__ . '/../common/bootstrap.php';`

- **언제 적합한가?**
	- 공통 유틸, 설정, 초기화 스크립트 등 **PHP 로직 파일**을 불러올 때.        
    - 현재 파일 위치와의 상대적 경로를 명확히 하고 싶을 때.        
    - 다른 서버/경로로 배포될 수 있어도 **폴더 구조만 같으면 안전하게 동작**해야 할 때.     
- **장점**    
    - 파일 구조가 바뀌지 않는 한 경로 문제 없음.        
    - 외부 상수(`BASE_PATH`) 의존성이 없어 독립적.        
- **적용 예시**    
    - `bootstrap.php`, `config.php`, `db.php`, `autoload.php` 같은 환경 설정 파일.        
    - `optimizer_user_ol.php`, `optimizer_ol.php` 같이 주문 로직만 담당하는 include 파일.        

---

## 2. `BASE_PATH` 기반 `require_once`

`<?php require_once BASE_PATH . '/footer.php'; ?>`

- **언제 적합한가?**    

    - 레이아웃 파일(header, footer, side_menu 등)을 HTML 안에서 불러올 때.        
    - 프로젝트 루트 기준으로 공용 UI 컴포넌트를 불러올 때.        
    - 경로가 HTML 구조와 분리되어 **일관되게 고정 루트에서 관리**될 때.     
    
- **장점**    

    - HTML 안에서 직관적 (header/footer 위치 고정).        
    - 어디서 호출해도 항상 같은 위치의 파일을 가져옴.    
    
- **적용 예시**    

    - `header.php`, `footer.php`, `side_menu.php` 등 레이아웃/템플릿 조각.        
    - 주문조회 페이지(`orderlist.php`, `orderdetail.php`)에서 UI 영역 삽입 시.        

---

## 3. 추천 패턴

- **비즈니스 로직 / 설정**    

    - `__DIR__` 사용 → 현재 파일 기준 상대경로가 가장 안전     
    
- **레이아웃 / 뷰**    

    - `BASE_PATH` 사용 → 프로젝트 루트 기준 절대경로로 불러오는 게 직관적        

---

👉 정리하면,

- **`orderdetail.php`, `orderlist.php`**:    
    - 상단: `require_once __DIR__ . '/../common/bootstrap.php';`        
    - 하단: `<?php require_once BASE_PATH . '/footer.php'; ?>`
        
- **optimizer_xxx.php 같은 로직 include**:    
    - 무조건 `__DIR__` 기반


## 따라서 경로 처리 원칙

1. **서버 내부 파일(include, require)**
    
    - `__DIR__` 또는 `BASE_PATH` 활용        
    - 예:        
        `require_once __DIR__ . '/../common/bootstrap.php'; require_once BASE_PATH . '/includes/db.php';`
        
2. **브라우저가 접근하는 리소스(JS, CSS, 이미지)**
    
    - `BASE_URL` 또는 `url_to()` 활용        
    - 예:        
```js
<script src="<?php echo url_to('/js/optimizer_user_js.php'); ?>"></script> <link rel="stylesheet" href="<?php echo url_to('/css/order.css'); ?>"> <img src="<?php echo url_to('/images/logo.png'); ?>" alt="Logo">
```

---

👉 정리하면,

- `bootstrap.php`에서 경로/URL 관련 상수를 세팅해두었기 때문에,    
- **PHP 내부 require/include → `__DIR__` 또는 `BASE_PATH`
    
- **HTML에서 외부 리소스 로드 → `url_to()` 또는 `BASE_URL`**
    
이렇게 나눠서 쓰면 경로 문제가 확실히 줄어듭니다.


## 4. `url_to()`

- **브라우저가 접근해야 하는 리소스** 경로를 만들 때 씁니다.
    
- 예시:
```js
    <script src="<?php echo url_to('/js/app.js'); ?>"></script> 
    <link rel="stylesheet" href="<?php echo url_to('/css/style.css'); ?>"> <img src="<?php echo url_to('/images/logo.png'); ?>" alt="로고">
```
    
    
- 결과: `https://example.com/js/app.js` 와 같은 URL.
    

---

## 5. `path_to()`

- **서버 내부에서 include/require, 파일 조작**을 할 때 씁니다.
    
- 예시:
```php
    require_once path_to('/common/bootstrap.php'); 
    $file = path_to('/storage/uploads/test.csv'); 
    file_put_contents($file, $data);
```
    
    
- 결과: `/var/www/project/common/bootstrap.php` 와 같은 **실제 파일 시스템 절대경로**.
    

---

✅ **차이 요약**

- `url_to()` → **웹 접근 경로 (브라우저)**
    
- `path_to()` → **서버 내부 파일 경로 (PHP)**
    

---

👉 지금까지 우리가 다룬 부분에 대입하면:

- `footer.php`, `header.php` 같은 include: **`path_to()`** 또는 `BASE_PATH`/`__DIR__`
    
- JS/CSS/이미지 로드: **`url_to()`**