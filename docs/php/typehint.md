---
layout: default
title: PHP 타입 힌트
parent: PHP
nav_order: 2
---

# 타입 힌트

`@var`는 **PHPDoc 문법**에서 쓰이는 **타입 힌트 주석**입니다.

---

## 기본 형태

`/** @var 타입 변수명 */`

예:

`/** @var PDO $dbh */`

---

## 의미

- **런타임에는 아무 효과 없음** → PHP 실행 시 무시됩니다.
    
- **IDE / 정적 분석기용 메타데이터**
    
    - “`$dbh` 변수는 `PDO` 타입이다” 라고 툴에 알려줍니다.
        
    - 그 결과 IDE가 `$dbh->` 를 입력했을 때 `prepare()`, `query()` 같은 PDO 메서드 자동완성을 보여줍니다.
        
    - 정적 분석기(phpstan, psalm 등)가 “`$dbh`는 PDO인데, 없는 메서드를 호출했네” 같은 에러를 잡아낼 수 있습니다.
        

---

## 예시

```php
/** @var User $user */
$user = getCurrentUser();
echo $user->getName(); // IDE가 getName() 자동완성 지원

```

```php
/** @var string[] $names */
$names = ['Alice', 'Bob'];
foreach ($names as $n) {
    echo strtoupper($n); // IDE가 $n을 string으로 인식
}

```

---

✅ 정리:

- `@var` = 타입 힌트 주석
    
- 실행에는 영향 없음
    
- IDE 자동완성 + 정적 분석 도움
    
- 복잡한 배열/객체 타입도 표현 가능 (`User[]`, `array<string,int>` 등)