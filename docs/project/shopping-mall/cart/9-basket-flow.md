---
layout: default
title: "[cart] 장바구니 로직 정리 상세"
parent: cart
nav_order: 9
---

# 🛒 basket.php 로직 상세 정리 (코드 흐름 + API 호출 순서)

## 1. 페이지 초기화 (View Layer)
- **PHP include**
  - `bootstrap.php`
  - `side_menu.php`, `header.php`, `footer.php`
- **JS & CSS 로드**
  - jQuery (noConflict → `$JQ` 별칭)
  - 공통 CSS (`optimizer.php`), 장바구니 CSS (`basket.css`)
  - 사용자 스크립트 (`optimizer_user_js.php`)

---

## 2. 초기 렌더링 & 데이터 로딩
- JS 즉시 실행 함수로 동작 시작
- **`refresh()` 호출**
  - `API("list")` → `../api/cart_proc.php?action=list`
  - 서버 응답: `{ ok:true, items:[...], subtotal, shipping, total }`
  - 화면 반영
    - 상품 리스트 → `#cart-rows`
    - 합계 → `#sum-subtotal`, `#normal_normal_ship_fee`, `#normal_normal_ship_fee_sum`
    - 서버 합계 캐시 → `LAST_TOTALS`

---

## 3. 합계 처리 로직
- **전체선택 상태**
  - 서버 응답(`LAST_TOTALS`) 그대로 표시
- **일부선택 상태**
  - 선택된 상품들의 `row_sum` 합산
  - 합계/총액에 반영 (배송비는 0 처리)

---

## 4. 이벤트 흐름 (주요 동작)

### ① 전체선택/해제
- 버튼 `#product_select_all` 클릭
- 체크박스 전체 토글 → `applySelectedTotals()` 호출

### ② 개별선택
- 체크박스 변경 이벤트 발생
- `syncSelectAllButton()`으로 버튼 상태 갱신
- `applySelectedTotals()`로 합계 반영

### ③ 수량 변경
- **증가/감소 버튼** (`.up`, `.down`)
- **직접 변경** (`.btn-apply`)
- 동작: `API("update", { product_id, qty })`
- 성공 시: `refresh()` + `$(document).trigger("cart:updated")`

### ④ 삭제
- **개별삭제** (`.btnDelete`) → `API("remove")`
- **선택삭제** (`#btn-delete-selected`) → `API("removeMany", { ids:[...] })`
- 성공 시: `refresh()` + `cart:updated` 트리거

### ⑤ 주문 버튼
- `#btn-order-all`, `#btn-order-selected` 클릭 시 `alert("연동 필요")`

---

## 5. API 호출 순서 정리

| 동작             | 호출 API                                     | 후처리 |
|------------------|---------------------------------------------|--------|
| 페이지 로딩      | `API('list')`                               | 상품목록/합계 렌더링 |
| 수량 변경        | `API('update', { product_id, qty })`        | refresh() + cart:updated |
| 개별상품 삭제    | `API('remove', { product_id })`             | refresh() + cart:updated |
| 선택상품 삭제    | `API('removeMany', { ids:[...] })`          | refresh() + cart:updated |

---

## 6. 이벤트 전파
- **`$(document).trigger("cart:updated")`**
  - `header.php` 등에서 이 이벤트를 수신해 **헤더 미니카트 수량 동기화**

---

## 7. 전체 흐름 요약 (시퀀스)

1. User → Browser: 장바구니 페이지 접속
2. Browser → Server: `list` API 호출
3. Server → Browser: 상품목록/합계 응답
4. Browser: 상품 UI 렌더링
5. User → Browser: 수량 변경/삭제/선택 조작
6. Browser → Server: `update/remove/removeMany` API 호출
7. Server → Browser: 처리 결과 반환
8. Browser: `refresh()` 실행, UI 업데이트
9. Browser: `cart:updated` 이벤트 발생 → Header 갱신

