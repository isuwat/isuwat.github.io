---
layout: default
title: "[code] 코드 스타일"
parent: Web
nav_order: 3
---

# KPP 쇼핑몰 프로젝트 – 코드 스타일 가이드 (PHP/JS/SQL)

본 문서는 한국프리페이드(온캐시) 쇼핑몰 프로젝트의 일관성과 가독성을 높이기 위한 **코드 스타일 규칙**을 정의합니다. 팀 합의 없이 임의 변경하지 않습니다.

---

## 0) 공통 원칙

- **가독성 우선**: 사람이 읽기 쉬운 코드가 최적화보다 중요합니다.
- **일관성 유지**: 동일한 문제는 동일한 방식으로 표현합니다.
- **자동화 도구 사용**: 포맷은 도구(Prettier, ESLint, PHP-CS-Fixer)로 자동 정리합니다.
- **Diff 친화적**: 불필요한 줄바꿈/트레일링 스페이스 금지.
- **주석은 의도 중심**: 코드가 *왜* 이렇게 되었는지 기록합니다(무엇/how는 코드로 충분해야 함).

---

## 1) PHP (PSR-12 기반)

### 1.1 파일/기본 규칙

- 인코딩 UTF-8, LF(Line Feed) 사용.
- `<?php` — PHP 오프닝 태그는 단일 사용; `?>`는 **순수 PHP 파일**에선 생략 가능.
- 파일 최상단 `declare(strict_types=1);`는 신규 코드에 권장.

### 1.2 들여쓰기/여백

- 들여쓰기: **4 spaces**.
- 한 줄 최대 120자 권장(넘으면 의미 단위로 줄바꿈).
- 연산자 주변, 콤마 뒤에 공백 1칸.
- 제어구문/함수 본문은 한 단계 들여쓰기.

### 1.3 중괄호/구문

```php
if ($cond) {
    // ...
} elseif ($other) {
    // ...
} else {
    // ...
}
```

- 함수/클래스/제어문 여는 중괄호는 같은 줄 끝.

### 1.4 네이밍

- 변수/함수: `snake_case` 또는 `lowerCamelCase` 중 **파일 내 일관성 유지** (신규 코드는 `lowerCamelCase` 권장).
- 클래스: `PascalCase`.
- 상수: `UPPER_SNAKE_CASE`.

### 1.5 문자열/배열/비교

- 문자열은 기본 `'single quote'` (변수 보간 필요 시 `"double"`).
- 배열은 짧은 문법 `[]` 사용.
- 비교: 값 비교는 `===`, 느슨한 비교 지양. 널 병합 `??` 적극 사용.

### 1.6 포함 경로(depth) 예시

> **단일 대입문은 들여쓰기 없이 좌측 정렬**(보편적). HTML 내부에서는 주변 들여쓰기에 맞춰 선택.

```php
$depth = str_repeat('..' . DIRECTORY_SEPARATOR, substr_count(dirname($_SERVER['SCRIPT_NAME']), '/'));
$depth = rtrim($depth, DIRECTORY_SEPARATOR) ?: '.';
```

### 1.7 에러 처리 / 보안

- 입력값 검증 필수(`filter_input`, `ctype_*`, 커스텀 validator).
- SQL은 **준비된 쿼리/바인딩** 사용.
- `$_SERVER['PHP_SELF']` 직접 출력 지양 (`SCRIPT_NAME`/`__FILE__` 기반 경로 계산 권장).

---

## 2) JavaScript / TypeScript (Airbnb + Prettier)

### 2.1 포맷/문법

- 들여쓰기: **2 spaces**.
- 세미콜론 **사용**.
- 따옴표: **single** (`'`), 필요 시 템플릿 리터럴.
- 엄격 모드/모듈(`type="module"` 또는 번들러) 선호.

### 2.2 변수/함수

- 변하지 않는 값은 `const`, 재할당은 `let`. `var` 금지.
- 함수는 화살표 함수 우선. 클래스/프로토타입은 필요한 경우에만.

### 2.3 네이밍

- `camelCase`, 클래스/컴포넌트는 `PascalCase`.
- DOM 셀렉터 캐시는 `$el` 접두어 선택 가능(팀 합의 시).

### 2.4 예시(장바구니 카운트)

```html
<script>
(function () {
  const $ = window.$JQ || jQuery;
  const CART_API_BASE = "<?php echo $depth; ?>/api/cart_proc.php";

  function refreshCartCount() {
    $.getJSON(CART_API_BASE, { action: 'list' })
      .done(function (res) {
        if (res.ok) {
          let totalQty = 0;
          (res.items || []).forEach((it) => (totalQty += it.quantity || 0));
          $('.xans_myshop_main_basket_cnt').text(totalQty);
        }
      })
      .fail(function (xhr) {
        console.warn('[cartCount] failed', xhr.responseText);
      });
  }

  $(document).ready(refreshCartCount);
})();
</script>
```

- **함수 본문 블록은 반드시 들여쓰기**.
- 화살표 함수 파라미터 한 개라도 괄호 유지 `(it) =>` (팀 일관성용).

### 2.5 기타 규칙

- \*\*엄격 비교 \*\*\`\`, 널 병합 `??`, 옵셔널 체이닝 `?.` 사용.
- Ajax/Fetch는 **에러 처리**(`.catch` 또는 `.fail`) 필수.
- Magic number 지양: 상수로 분리.

---

## 3) HTML / CSS

### 3.1 HTML

- 속성 순서: `class`, `id`, `name`, `data-*`, `src/href`, `type`, `aria-*`, `role`.
- boolean 속성은 축약(`<input disabled>`).
- `data-*`는 **kebab-case**.

### 3.2 CSS (또는 SCSS)

- 들여쓰기 2 spaces, 탭 금지.
- 클래스는 **kebab-case**. BEM 권장: `block__element--modifier`.
- 공통 색/spacing/폰트는 변수/유틸 클래스로 관리.

---

## 4) SQL (MySQL 8 기준)

### 4.1 포맷

- 키워드 대문자(`SELECT`, `FROM`, `WHERE`).
- 컬럼/테이블은 소문자 + `snake_case`.
- **JOIN은 명시적**(`JOIN ... ON ...`) 사용.
- 긴 리스트는 행바꿈 정렬:

```sql
SELECT
    p.product_id,
    p.product_name,
    COALESCE(p.final_price, p.sale_price, p.buy_price, 0) AS effective_price
FROM shop_products AS p
WHERE p.display = '1'
  AND p.stat <> '2';
```

### 4.2 이름 규칙

- 테이블: `shop_orders`, `shop_order_product` 등 **도메인\_주어** 형태.
- PK: `id` 또는 `seq_no`(기존 스키마 유지). FK는 `*_id`/`*_seq_no`.
- 인덱스: `ix_<table>_<cols>`.
- 뷰: `v_*` 접두어 (예: `v_products_effective`).
- 스토어드 프로시저: 동사 기반 `add_*`, `select_*`, `*_req`, `*_rsp` 등.

### 4.3 트랜잭션/에러

- `START TRANSACTION; ... COMMIT;` 범위 최소화.
- 예외 핸들러에서 **롤백 + 로깅**. 사용자 메시지와 시스템 메시지 구분.

---

## 5) 주석/문서화

- **파일 헤더**: 목적/작성자/변경이력(필요한 경우) 간단히.
- **함수 DocBlock**: 파라미터/리턴/예외.
- **중요 로직**: 의도, 제약사항, 성능 고려사항.

예)

```php
/**
 * 장바구니에 상품 추가(존재시 수량 가산)
 * @param string $userId
 * @param string $productId
 * @param int    $qty 기본 1
 * @throws RuntimeException
 */
```

---

## 6) Git 커밋/브랜치

- 브랜치: `feature/…`, `fix/…`, `hotfix/…`, `chore/…`.
- 커밋 메시지(영문 권장):
  - 제목(50자 내): 동사 원형 – 예) `Add cart count refresher on header`
  - 본문(선택): 배경/제약/이슈 링크.
- PR 템플릿: 변경 요약, 테스트 방법, 스크린샷, 영향 범위.

---

## 7) 도구 설정(권장)

- **Prettier**: JS/TS/HTML/CSS 자동 포맷.
- **ESLint**: `eslint:recommended` + `airbnb-base`. Rule 예) `no-var`, `eqeqeq`, `no-console`(필요 파일만 허용).
- **PHP-CS-Fixer**: PSR-12 preset.
- **EditorConfig**: `indent_size`, `end_of_line=lf`, `insert_final_newline=true`.

`.editorconfig` 예시

```
root = true

[*]
charset = utf-8
end_of_line = lf
insert_final_newline = true
indent_style = space
indent_size = 2

[*.php]
indent_size = 4
```

---

## 8) 예외/레거시

- 기존 파일 스타일을 대거 변경 시 **리뷰어 합의 후** 진행(대규모 diff 방지).
- 레거시와 신규가 혼재된 경우: **신규/수정 구간만** 규칙을 적용해 점진 정착.

---

## 9) 체크리스트 (PR 전)

-

---

### 부록) 스니펫 모음

- **PHP depth 계산(동적)**

```php
$depth = str_repeat('..' . DIRECTORY_SEPARATOR, substr_count(dirname($_SERVER['SCRIPT_NAME']), '/'));
$depth = rtrim($depth, DIRECTORY_SEPARATOR) ?: '.';
```

- **JS Ajax + 에러 처리 패턴**

```js
fetch(`${API}/cart?action=list`)
  .then((r) => r.ok ? r.json() : Promise.reject(r))
  .then(({ ok, items = [] }) => ok && updateUI(items))
  .catch((e) => console.warn('[cart] failed', e));
```

> 문의/추가 제안은 코드리뷰에서 자유롭게 남겨주세요.

