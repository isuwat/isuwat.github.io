---
layout: default
title: "setcookie()"
parent: php
nav_order: 11
---


### 서브도메인(subdomain)
**서브도메인(subdomain)**은 `example.com`의 **왼쪽**에 붙는 레이블입니다. DNS는 **오른쪽이 상위, 왼쪽으로 갈수록 하위** 구조라서 그래요.

#### 한눈에 정리

```bash
scheme://host[:port]/path
host = [sub2.][sub1.]domain.tld

```

- `example.com` → (루트/에이펙스 도메인)    
- `a.example.com`, `b.example.com` → **example.com의 서브도메인**    
- `a.b.example.com` → 더 깊은 서브도메인    
- `www.example.com` → 흔한 서브도메인 예시    

### 헷갈리기 쉬운 것

- `example.com/a` → **호스트는 `example.com`**, `/a`는 **경로(path)**. 서브도메인 아님.    
- `example.com.b` → TLD가 `b`인 **전혀 다른 도메인**(실존 여부와 무관). `example.com`의 서브도메인 아님.    

### 쿠키와의 연결

- `Domain=example.com` → `example.com` **및** `a.example.com`, `a.b.example.com` 같은 **모든 하위 호스트**로 전송.    
- `Path=/` vs `Path=/a` → 쿠키가 **어떤 경로에만** 붙을지 결정(도메인과는 별개 규칙).    

즉, “서브”가 뒤에 온다는 뜻이 아니라 **상위 도메인의 ‘하위’(왼쪽)로 붙는다**는 의미라고 이해하시면 됩니다.

### setcookie()

**쿠키는 “서버→클라이언트 응답” 때 한 번 내려보내 저장시키고, 그다음부터는 “클라이언트→서버 요청”에 자동으로 실려 옵니다.**  
즉, `setcookie(domain=example.com)`을 호출한 **그 응답**에 `Set-Cookie` 헤더가 담겨 가고, **이후 요청들**에서 브라우저가 `Cookie` 헤더로 붙여 보냅니다.

### 흐름 요약

1. **첫 요청**

```scss
클라이언트  ──(GET /, Host: example.com)──▶  서버(PHP: setcookie 호출)

```

2. 서버 응답(쿠키 저장 지시)

``` scss
서버 ──(HTTP 200
        Set-Cookie: name=value; Domain=example.com; Path=/; Expires=...
        ...)──▶ 브라우저   ← 이때 브라우저가 쿠키를 저장

```

3. 다음 요청부터

```scss
브라우저 ──(GET /, Host: example.com
            Cookie: name=value)──▶ 서버

```

### 핵심 포인트

- `Set-Cookie`는 **응답 헤더**입니다(서버가 “저장해!”라고 지시).    
- 저장된 뒤에는 **요청 헤더 `Cookie`**로 브라우저가 자동 첨부합니다(도메인/경로/SameSite 규칙이 맞을 때).    
- 서버는 필요할 때만 다시 `Set-Cookie`를 보내 **갱신/삭제**합니다(매 응답마다 자동 전송되는 것 아님).    
- `Domain=example.com`이면 `example.com` **및** 그 **서브도메인들**(`a.example.com`, `b.example.com` 등)로의 요청에 붙습니다. (호스트 전용으로 제한하려면 `domain`을 생략)


### 쿠키 보관 기한”을 설정

다음처럼 `setcookie()`의 `expires`(만료 시각) 옵션으로 “쿠키 보관 기한”을 설정합니다.  
(PHP 7.3+ 기준: 옵션 배열 권장)

```php
// 1) 7일 동안 유지되는 쿠키
setcookie('user_id', '42', [
  'expires'  => time() + 60*60*24*7,  // 지금부터 7일
  'path'     => '/',                  // 전체 경로
  'secure'   => true,                 // HTTPS에서만 전송
  'httponly' => true,                 // JS에서 접근 불가
  'samesite' => 'Lax',                // CSRF 완화 (Lax/Strict/None)
]);

```

### 핵심 정리

- **세션 쿠키(브라우저 닫으면 삭제)**: `expires`를 생략하거나 `0`으로 설정.

```php
setcookie('temp', 'ok', ['path' => '/', 'httponly' => true]);

```

- **특정 기간 유지**: `time() + 초`로 만료 시각 계산.    
    - 1일: `time() + 60*60*24`        
    - 30일: `time() + 60*60*24*30`

- **DateTime 사용**:

```php
$dt = new DateTime('+30 days');
setcookie('token', 'abc', ['expires' => $dt->getTimestamp(), 'path' => '/']);

```

**삭제(즉시 만료)**: 과거 시간으로 재설정

> 주의: **같은 path/domain**으로 설정해야 정확히 지워집니다.

```php
setcookie('user_id', '', [
  'expires' => time() - 3600,
  'path'    => '/',
]);

```

- **SameSite 주의**:    
    - `None`을 쓰면 반드시 `secure => true` 필요(HTTPS).        
    - 일반 웹은 `Lax`가 기본적으로 무난.        

### 자주 하는 실수

- `setcookie()`는 **헤더 전송**이므로 **어떤 출력도 하기 전에** 호출해야 합니다(공백/echo/HTML 출력 전).    
- 도메인/경로 불일치로 삭제가 안 되는 경우가 많습니다. 생성할 때와 **동일한 `path`/`domain`** 사용하세요.


