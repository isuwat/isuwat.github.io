---
layout: default
title: PHP session_start()
parent: PHP
nav_order: 9
---


# session_start()

질문:session_start(); 함수는 require_once로 공통 포함 파일에 존재하면 안되겠네?

`session_start();`는 PHP에서 **세션을 초기화**하는 함수인데, 중요한 제약이 있습니다:

- 같은 요청(Request) 안에서 **한 번만 호출 가능**합니다.
    
- 이미 세션이 시작된 상태에서 또 호출하면
    
    - PHP 5.x: `Notice: A session had already been started`
        
    - PHP 7.x~8.x: `Warning: session_start(): Session already started`  
        가 발생합니다.
        

---

### 그래서 왜 `require_once`에 두면 안 되냐?

- `require_once`는 **파일 로딩은 1회만 보장**하지만,  
    **다른 진입점(페이지)**에서 또 그 파일을 불러올 수는 있습니다.
    
- 예를 들어:
```php
// common.php
<?php
session_start();

```

```php
// pageA.php
require_once 'common.php';

// pageB.php
require_once 'common.php';

```

- `pageA.php`와 `pageB.php` 각각은 **별도 요청**이라서 결국 `common.php`가 실행되고 `session_start()`가 또 호출됩니다.
    
- 이렇게 되면 매 요청마다 무조건 `session_start()`가 실행 → 충돌은 아니지만,  
    이미 세션이 열린 상태에서 불필요하게 또 실행될 가능성이 큼.
    

---

### 안전하게 쓰는 방법 ✅

1. **조건 체크 후 실행**
```php
if (session_status() === PHP_SESSION_NONE) {
    session_start();
}
```

2. 또는 **세션이 꼭 필요한 특정 위치(예: index.php, 로그인 진입점 등)** 에서만 명시적으로 `session_start()` 호출.
    
3. 공통 include 파일(common.php)에는 `session_start();`를 직접 두지 않고,  
    세션 제어 전용 bootstrap/init 파일을 따로 두는 방식도 많이 씁니다.
    

---

👉 정리하면:

- `require_once`가 중복 로딩은 막아주지만, **세션은 HTTP 요청 단위로 초기화**되기 때문에 매번 실행됨.
    
- 따라서 `session_start()`는 공통 포함 파일에 그냥 두지 말고, **조건 체크**를 붙여 두거나,  
    **세션 시작 지점에서만 명시적으로 호출**하는 게 안전합니다.