---
layout: default
title: "[cart] 카테고리 메뉴 출력 이슈"
parent: shop
nav_order: 16
---

## basket.php 카테고리 메뉴 출력 문제 원인 & 해결

### 1. 원인

- `header.php`는 `categorymenu_shop` 네비게이션을 정상적으로 출력함.
- 그러나 `basket.css`에서 아래와 같이 강제로 `display:none;` 처리되어 있었음:
  ```css
  #categorymenu_community {
      display: none;
  }

  #categorymenu_shop {
      display: none;
  }
  ```
- 이 때문에 DOM은 생성되었지만 화면에 보이지 않았음.

### 2. 해결 방법

#### (A) 불필요한 숨김 규칙 삭제

- `basket.css`에서 `#categorymenu_shop { display:none; }`를 제거.
- community 메뉴만 숨기려 했다면 `#categorymenu_community`만 남김.

#### (B) override 적용

- basket.css는 유지하면서 SHOP 메뉴를 보이게 하고 싶다면, 아래 덮어쓰기 규칙 추가:
  ```css
  #categorymenu_shop {
      display: flex !important;  /* flex 컨테이너로 강제 */
      justify-content: flex-start;
      overflow: visible;
  }
  #categorymenu_shop li:first-child {
      display: list-item !important;
      visibility: visible !important;
      margin-left: 0 !important;
      transform: none !important;
  }
  ```

#### (C) 공통 CSS로 이관

- nav 관련 스타일은 `list.css` 또는 `optimizer.css` 같은 공통 파일로 이동.
- basket.css에는 nav를 숨기는 규칙을 두지 않음.

### 3. 정리

- **원인:** basket.css에서 `#categorymenu_shop`을 숨겼기 때문.
- **해결:** 숨김 규칙 삭제하거나 override로 보이게 조정.
- **추천:** nav 스타일을 공통 CSS로 이관하고, basket.css는 장바구니 전용 스타일만 유지.

