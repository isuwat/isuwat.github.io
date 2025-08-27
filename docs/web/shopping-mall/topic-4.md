---
layout: default
title: "[cart] 장바구니 로직 정리"
parent: shop
grand_parent: Web
nav_order: 4
---

# 장바구니 연동 로직 정리

## 1) 데이터 흐름

1. 페이지 진입 (`basket.php`)  
   → `API("list")` 호출 → `cart_proc.php?action=list`  
   → 프로시저 `select_shop_cart_total_list(p_user_id)` 실행  
   → **ResultSet#1:** 아이템 목록 / **ResultSet#2:** 합계(총상품금액, 배송비, 할인, 총합)  
   → JSON 응답 → 화면 렌더/합계 표시.

2. 수량 증감/변경  
   → `API("update", {product_id, qty})` → `update_shop_cart` 호출  
   → 성공 시 `refresh()` 재호출(목록+합계 재조회) → 화면 갱신.

3. 단일 삭제  
   → `API("remove", {product_id})` → `remove_shop_cart` 호출  
   → 성공 시 `refresh()`.

4. 선택 삭제  
   → 체크된 행들의 `product_id` 수집 → `API("removeMany", {ids: JSON.stringify([...])})`  
   → 각 id 별 `remove_shop_cart` 호출 → 결과 모아 응답  
   → 성공 시 `refresh()`.

---

## 2) 세션 / 사용자 ID 정책

- 공통: 세션 시작 후 `$_SESSION['user_id']` 점검
  - 비어있거나 길이 20 초과 시: `guest_` + `substr(session_id(), 0, 14)` 로 대체
  - 항상 **최대 20자** 보장 (DB 스키마 `VARCHAR(20)`)
- (주의) 테스트용 하드코딩 `'$user_id = "isuwat2024";'`는 실제 운영 시 제거 필요.

---

## 3) API 규격 (cart_proc.php)

- `list` (POST)
  - 응답:  
    - `items[]`: `product_id, product_name, effective_price, quantity, row_sum, image_url(또는 product_thumb_img_url/product_img_url)`  
    - `subtotal, shipping, discount, total`
- `add` (POST: `product_id, qty`) → `add_shop_cart`
- `update` (POST: `product_id(or id), qty`) → `update_shop_cart`
- `remove` (POST: `product_id(or id)`) → `remove_shop_cart`
- `removeMany` (POST: `ids` = JSON 배열 문자열) → 각 항목 `remove_shop_cart` 반복 실행
- `clear` → `clear_shop_cart`

각 액션 호출 후 `@rc`, `@msg` 조회하여 `{ok:true}` 또는 `{ok:false, code, msg}`로 반환.

---

## 4) 프런트 로직 (basket.php)

### 4.1 Ajax 래퍼
```js
const API = (action, data={}) => $.ajax({
  url: "../api/cart_proc.php?action=" + action,
  method: "POST",
  data,
  dataType: "json",
  xhrFields: { withCredentials: true }
});
```

### 4.2 렌더링/합계 갱신

* refresh()
  * list 응답으로 행 템플릿 렌더
  * 좌측 요약(상품금액/배송비/합계) 표시
  * 우측 최종요약(총상품금액/총배송비/결제예정금액) 표시
  * 서버 합계 캐시 LAST_TOTALS 저장
  * 전체선택 버튼 상태 동기화 → 선택합계 힌트/요약 덮어쓰기 실행

### 4.3 체크박스/전체선택 & 합계
* 개별 체크:
  * syncSelectAllButton() → 버튼 라벨/상태 동기화
  * calcSelectedSumHint() → “주문상품 (선택합계: N원)” 표시
  * updateSelectedTotals() → 좌측 요약을 선택 기준으로 덮어쓰기
  * applySelectedTotals() → 우측 요약도 선택 기준으로 덮어쓰기(전체 선택이면 서버 합계 그대로, 일부만 선택이면 배송비 0으로)

* 전체선택 버튼:
  * 클릭 시 전체 체크 상태 토글 → 위 3개 함수로 동일하게 반영.

### 4.4 수량/삭제 이벤트

* 증감/변경: update 호출 후 refresh()
* 단일 삭제: remove 호출 후 refresh()
* 선택 삭제:
  * 체크된 행 id → 중복 제거(Set) → 확인창 → removeMany 호출
  * 성공 시 refresh() / 실패 시 메시지

### 5) 삭제 트랜잭션/커서 처리
단일 삭제
* CALL remove_shop_cart(:uid, :pid, @rc, @msg) 실행 → SELECT @rc, @msg
* 반환 후 {ok:true} 또는 에러.

선택 삭제
* (현재 버전: 비트윈형 처리)
  * 각 pid에 대해 CALL remove_shop_cart(...) 실행 → 결과 조회
  * 성공 목록/실패 목록 모아 부분 실패도 감지
  * 모두 성공 시 {ok:true}
  * 일부 실패 시 {ok:false, code:100, fails:[{pid, rc, msg},...]}

중요: MySQL의 프로시저 호출 뒤에는 커서/Rowset을 적절히 정리해야 합니다.
루프 내에서 prepare()->execute() 이후
```php
while ($st->nextRowset()) { /* flush */ }
$st->closeCursor();
```
후속 SELECT @rc, @msg를 실행하세요.

### 6) 이미지/링크 키 정합
* 서버는 호환을 위해 다음처럼 내려줌:
  * product_thumb_img_url, product_img_url, image_url

* 프런트 템플릿에서는 우선순위로 사용:
  * it.image_url || it.product_thumb_img_url || it.product_img_url || ''  
  (현재 코드는 image_url 우선 사용. 서버에서 맞춰 내려주므로 OK)