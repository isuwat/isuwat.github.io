---
layout: default
title: PHP include require
parent: PHP
nav_order: 8
---



# include require

PHP에서 비슷한 함수가 네 가지 있는데 각각 동작이 달라요.

|함수|중복 호출 시 동작|에러 처리 방식|
|---|---|---|
|`include 'file.php'`|여러 번 불러옴 (중복 가능)|파일 없으면 **warning** 발생 후 계속 실행|
|`include_once 'file.php'`|한 번만 불러옴 (이미 불렀으면 무시)|파일 없으면 **warning** 발생 후 계속 실행|
|`require 'file.php'`|여러 번 불러옴 (중복 가능)|파일 없으면 **fatal error** 발생 후 실행 중단|
|`require_once 'file.php'`|한 번만 불러옴 (이미 불렀으면 무시)|파일 없으면 **fatal error** 발생 후 실행 중단|

---

👉 따라서

- **`require_once '파일.php';`** 는 **다른 페이지에서 같은 파일을 또 참조해도 "한 번만" 불러옵니다.**
    
- PHP 내부적으로 **절대 경로 기준**으로 체크하기 때문에, 동일한 파일을 `require_once` 했다면 두 번째 이후 호출은 무시돼요.
    

즉, 공통 함수/설정/클래스 파일은 보통 `require_once`를 사용해서 **중복 로딩으로 인한 함수 재정의 에러**를 막습니다.