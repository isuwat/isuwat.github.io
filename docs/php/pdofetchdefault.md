---
layout: default
title: PHP pdo fetch 기본값
parent: PHP
nav_order: 5
---


# pdo fetch 기본값 세팅

```php
$member = $st->fetch(PDO::FETCH_ASSOC) ?: [
    'cust_no' => null, 'user_cash' => 0,
    'receiver_name' => null, 'receiver_phone' => null,
    'post_code' => null, 'address' => null, 'address_detail' => null,
    'delivery_msg' => null
];

```

1. `$st->fetch(PDO::FETCH_ASSOC)`
    
    - PDOStatement 객체 `$st`에서 결과 행을 1개 가져옵니다.
        
    - 있으면 `['cust_no' => ..., 'user_cash' => ..., ...]` 형태의 **연관배열**을 반환.
        
    - 없으면 `false` 반환.
        
2. `?:` (null 병합은 아니고, **ternary shorthand**)
    
    - 왼쪽 값이 **truthy**(배열이 반환됨) → 그 값을 `$member`에 대입.
        
    - 왼쪽 값이 **falsy**(`false`, `null`, `0`, `''`) → 오른쪽 배열을 `$member`에 대입.
        

---

### 결과

- **조회 결과 있음** → `$member`에 DB에서 가져온 한 행이 들어감.
    
- **조회 결과 없음** → `$member`에 기본 구조 배열이 들어감:

```php
[
  'cust_no' => null,
  'user_cash' => 0,
  'receiver_name' => null,
  'receiver_phone' => null,
  'post_code' => null,
  'address' => null,
  'address_detail' => null,
  'delivery_msg' => null,
]

```
   

---

### 왜 쓰는가?

- 뷰에서 `$member['receiver_name']` 같은 값을 바로 출력할 때 **Undefined index 경고 방지**.
    
- 항상 같은 키 구조를 보장해서 코드가 단순해짐.
    

---

✅ 정리:  
이 코드는 **회원/배송지 조회 결과가 없을 경우에도 기본 키/값 구조를 채워서 `$member` 변수를 안전하게 쓸 수 있게** 해주는 방어 코드입니다.