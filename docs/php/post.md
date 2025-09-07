---
layout: default
title: PHP post 검증
parent: PHP
nav_order: 6
---


# POST 검증

PHP에서 `$_POST['scope']` 값이 올바른지 체크하는 **가장 기본적이고 안전한 방법**이에요 👍

```php
$scope = $_POST['scope'] ?? null;

if (!in_array($scope, ['direct', 'cart_all', 'cart_selected'])) {
    throw new Exception("Invalid purchase scope. Allowed values are: direct, cart_all, cart_selected.");
}

```
- `in_array()` → `$scope` 값이 배열에 포함되는지 확인
    
- 포함되지 않으면 예외 발생

## 📌 보완할 수 있는 부분

1. **엄격 모드 (`strict` 파라미터 true)**
    
    - 기본 `in_array()`는 타입 변환을 허용합니다.
        
    - 예: `"0"` == 0, `"false"` == false 로 처리될 수 있음.
        
    - 따라서 **문자열 일치만 허용**하려면 strict 모드를 켜는 게 좋아요:

```php
if (!in_array($scope, ['direct', 'cart_all', 'cart_selected'], true)) {
    throw new Exception("Invalid purchase scope. Allowed values are: direct, cart_all, cart_selected.");
}

```


    
2. **입력값 정리 (공백 제거)**
    
    - 혹시 `$_POST['scope']`에 앞뒤 공백이 들어올 수도 있으니 `trim()` 해주는 게 안전합니다.
        
```php
$scope = isset($_POST['scope']) ? trim($_POST['scope']) : null;

```
    
    
3. **NULL/빈값 처리**
    
    - 빈 값이나 미전송된 경우도 예외로 처리 가능:
        
```php
if ($scope === null || $scope === '') {
    throw new Exception("Missing scope parameter.");
}

```
   

---

## ✅ 최종 권장 예시

```php
$scope = isset($_POST['scope']) ? trim($_POST['scope']) : null;

if (!in_array($scope, ['direct', 'cart_all', 'cart_selected'], true)) {
    throw new Exception("Invalid purchase scope. Allowed values are: direct, cart_all, cart_selected.");
}

```

