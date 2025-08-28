---
layout: default
title: PHP 네이밍 가이드
parent: PHP
nav_order: 1
---

# PHP & 일반 프로그래밍 네이밍 컨벤션

## 1. 네이밍 스타일 종류



|스타일|설명|예시|
|---|---------------|---|
|snake_case|모든 단어를 소문자로,  `$father_jim`,  언더스코어(`_`)로 `$user_name`, 구분|`$order_id`|
|camelCase|첫 단어는 소문자,`$fatherJim`,이후 단어의 첫 글자는   `$userName`,대문자|`$orderId`|
|PascalCase|모든 단어의 첫 글자를`FatherJim`,대문자로 시작|`UserName`, `OrderId`|
|kebab-case|단어 사이를`father-jim`,하이픈(`-`)으로 구분|`user-name`, (CSS, URL 등에서 주로`order-id용사용)|                 


---
## 2. PHP에서의 권장 규칙 (PSR-1 / PSR-12 기준)

-   **클래스(Class) 이름** → `PascalCase`

    ``` php
    class ShoppingCart {}
    class OrderHistory {}
    ```

-   **메서드(Method) 이름** → `camelCase`

    ``` php
    public function addProduct($productId) {}
    public function getOrderList() {}
    ```

-   **변수(Variable) 이름** → `snake_case` (일반 PHP 커뮤니티 관례)

    ``` php
    $father_jim = "데이터";
    $user_name = "홍길동";
    $order_id = 1234;
    ```

-   **상수(Constant) 이름** → 대문자 + 스네이크 케이스

    ``` php
    const API_VERSION = '1.0';
    const MAX_RETRY_COUNT = 5;
    ```

---

## 3. 실무 예시

``` php
<?php
// 클래스명은 PascalCase
class UserProfile {
    // 속성(변수)은 snake_case
    private $user_id;
    private $user_name;

    // 메서드는 camelCase
    public function getUserName() {
        return $this->user_name;
    }

    public function setUserName($new_name) {
        $this->user_name = $new_name;
    }
}

// 상수는 대문자 스네이크 케이스
define('MAX_UPLOAD_SIZE', 1048576);

// 사용 예시
$user = new UserProfile();
$user->setUserName("홍길동");
echo $user->getUserName();
```

------------------------------------------------------------------------

## 4. 언제 어떤 케이스를 쓰면 좋은가?

-   **snake_case** → PHP 변수, SQL 컬럼명 (`user_id`, `order_id`)\
-   **camelCase** → JavaScript 함수, PHP 메서드 (`addProduct`,
    `getOrderList`)\
-   **PascalCase** → 클래스/타입 이름 (`UserProfile`, `OrderHistory`)\
-   **UPPER_SNAKE_CASE** → 상수 (`MAX_UPLOAD_SIZE`, `API_KEY`)\
-   **kebab-case** → CSS 클래스명, URL, 파일명 (`main-header`,
    `order-list`)

------------------------------------------------------------------------

✅ 이렇게 정리하면 GitHub Pages에서 그대로 보여줄 수 있습니다.