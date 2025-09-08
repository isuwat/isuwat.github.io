---
layout: default
title: "[cart] 주문-바로구매 장바구니 통합 처리 산출 요약"
parent: shop
nav_order: 26
---

# 주요 산출물(변경 사항)

### 1) 바로구매 (detail.php)

- 동작: 상품 상세에서 `add` 호출 → `cart_id` 수신 → 해당 `cart_id`만 주문서로 POST
    
- 구현 포인트:
    
    - `set_direct_buy()`
        
        - `$.ajax('../api/cart_proc.php', { method:'POST', data:{ action:'add', product_id, qty, options: JSON.stringify({...}) } })`
            
        - 성공 시 `res.cart_id`, `res.qty_after` 활용
            
        - 주문서 이동 POST 파라미터:
            
            - `mode=direct_buy`
                
            - `scope=selected`
                
            - `selected_cart_ids[]=<cart_id>`
                

### 2) 장바구니 전체구매 / 선택구매 (basket.php)

- 동작:
    
    - **전체구매**: 화면에 렌더된 모든 행에서 `{id: product_id, qty: quantity}` 수집 → 주문서로 POST
        
    - **선택구매**: 체크된 행만 수집 → 주문서로 POST
        
- 구현 포인트:
    
    - 각 행에 `data-id="<product_id>"` + `data-cart-id="<seq_no>"`(있으면) 유지
        
    - `pickItem($row)` → `{ id: product_id, qty: quantity }` 반환
        
    - `collectAll()` / `collectSelected()` → `[{id, qty}, ...]`
        
    - `postToOrderForm(products, {mode, scope})`에서 `product_data`(JSON)와 `mode`, `scope`를 숨김필드로 전달
        
    - 자잘한 수정:
        
        - `if (!products.lengh)` 오타 → `length`
            
        - 수량 변경 버튼 핸들러 누락분 보강
            
        - 합계표시(선택/전체)에 따른 표시값 동기화
            

### 3) cart_proc.php (API)

- **add**: 장바구니 담기 시 프로시저 `add_shop_cart` 호출 구조로 변경
    
    - IN: `p_user_id, p_product_id, p_qty, p_option_hash`
        
    - OUT: `po_cart_id`(=seq_no), `po_qty_after`, `po_result_code`, `po_err_msg`
        
    - 응답 JSON 예: `{ ok: true, cart_id: <int>, qty_after: <int> }`
        
- 참고:
    
    - 옵션 없는 경우 `option_hash='__noopt__'`로 통일
        
    - MySQL 8 경고 대응: `ON DUPLICATE KEY UPDATE`에서 `VALUES()` 대신 alias 사용
        
    - 이미 같은 `(user_id, product_id, option_hash)` 존재 시 수량 가산 후 해당 row의 `cart_id` 반환
        

### 4) order_form.php

- **입력**
    
    - `selected_cart_ids[]`(배열): 바로구매/선택구매에서 cart_id 직접 전달
        
    - `product_data`(JSON): 장바구니에서 화면 수집(또는 바로구매 포맷과 동일) — **없을 경우 방어 기본값 `"[]"` 적용**
        
- **조회 프로시저**
    
    - `shop_cart_selected_order_form(user_id, cart_ids_json, @rc, @msg)` (또는 `select_shop_cart_selected_list`)
        
    - 결과셋 구조:
        
        1. **회원/기본배송지**: `cust_no, user_cash, receiver_name, receiver_phone, post_code, address, address_detail, delivery_msg`
            
        2. **아이템 목록**: `cart_id, product_id, product_name, final_price(할인적용), sale_price, dis_rate(%), quantity, row_sum, image_url, ...`
            
        3. **합계**: `subtotal, shipping, discount(0), total`
            
- **PHP 처리**
    
    - `selected_cart_ids` → `array_map('intval', ...)` → `json_encode([...])` → `CALL ... CAST(:ids AS JSON)`
        
    - `nextRowset()`으로 3개 result set 파싱
        
    - `$products_info`로부터 `product_data`를 수동 생성:  
        `[{id: product_id, qty: quantity}, ...]` → `json_encode(...)`
        
    - **뷰 전달 시 방어**:  
        `<?= $product_data ?? "[]" ?>` 로 기본값 유지
        

### 5) DB: 프로시저/뷰

- **add_shop_cart** (업데이트)
    
    - `INSERT ... ON DUPLICATE KEY UPDATE` 시 alias 사용
        
    - `po_cart_id = LAST_INSERT_ID()`(중복/신규 모두 커버) + `po_qty_after` 조회
        
- **update_shop_cart**
    
    - `UPDATE` 후에는 `LAST_INSERT_ID()` 신뢰 불가 → OUT 파라미터에 `p_cart_id` 그대로 반환하거나, `SELECT ... WHERE seq_no = p_cart_id`
        
- **주문서 조회 프로시저**
    
    - JSON 배열을 `JSON_TABLE`로 파싱해 `selected_ids` 구성 (MySQL 8.0+)
        
    - 동일 CTE를 결과셋 #2, #3에서 반복 사용 시 성능 고려해 **임시테이블**(tmp_selected_ids, tmp_cart_items) 사용 권장
        
    - 배송비 계산 로직:
        
        - `grouped` = 정책/묶음 기준 그룹핑 (`delivery_policy_id, type, base_cost, free_over, combine_group_id`)
            
        - `shipping_calc` = 정책 유형별 합산 (free/fixed/conditional/area)
            
        - `subtotal_calc` = `SUM(final_price * quantity)`
            
- **뷰 v_products_effective**
    
    - `final_price` 물리 칼럼 제거 시:
        
        - `sale_price`와 `discount_rate`로 계산:  
            `ROUND(p.sale_price * (1 - COALESCE(p.discount_rate, 0))) AS final_price`
            
        - MySQL이 `ROUND(expr)`를 `ROUND(expr, 0)`로 표기 변경하는 건 정상
            
    - `effective_price`가 필요 없으면 단순화 가능
        

---

## 체크리스트 / TODO

-  cart_proc.php `add` 응답 스키마 확정: `{ok, cart_id, qty_after}`
    
-  detail.php `set_direct_buy()`에서 오류/중복 클릭 방지 및 실패 처리
    
-  basket.php: `data-cart-id` 포함 여부 점검(개별 삭제/업데이트 시 유용)
    
-  order_form.php: `product_data` 기본값 `"[]"` 방어, `selected_cart_ids` 미존재 시 에러 메시지
    
-  프로시저: JSON_TABLE 사용 시 MySQL 8.0.4+ 확인, 대안(임시테이블/IN 바인딩) 마련
    
-  배송비 정책 파라미터(`type, base_cost, free_over, combine_group_id`) 소스 일관화
    
-  QA:
    
    - 바로구매로 옵션/수량 케이스(신규 row, 기존 row 가산)
        
    - 장바구니 선택구매(선택/해제/수량변경 즉시 반영)
        
    - 합계/배송비 산출 정확성(임계값 free_over 경계)





# 바로구매 장바구니 주문 통합 처리 요약

**[EPIC] 스프린트6 – 장바구니 연동**  
**Goal:** 바로구매, 장바구니 전체구매, 장바구니 선택구매 플로우를 통합하고, cart_proc.php와 order_form.php를 통한 cart_id 기반 조회로 주문 프로세스를 안정화한다.

---

# Stories & Tasks

### Story 1. 바로구매 기능 개선

- **Task 1.1** detail.php → `set_direct_buy()` 수정
    
    - 상품 상세에서 cart_proc.php `add` 호출 → cart_id/qty_after 리턴 처리
        
    - 주문서 이동 시 POST: `mode=direct_buy`, `scope=selected`, `selected_cart_ids[]=cart_id`
        
- **Task 1.2** 옵션/수량 선택 처리 반영
    
    - option_hash 기본값 `__noopt__` 적용
        
    - 옵션 값 전달 구조 정리
        

---

### Story 2. 장바구니 전체구매/선택구매 연동

- **Task 2.1** basket.php → `collectAll()`, `collectSelected()` 구현
    
    - UI에 표시된 상품행에서 `{id, qty}` 추출
        
    - 체크박스 상태에 따라 선택상품만 수집
        
- **Task 2.2** 주문서 POST 전송
    
    - `postToOrderForm()`에서 product_data JSON 생성
        
    - `mode=cart_json_all | cart_json_selected`, `scope` 값 설정
        
- **Task 2.3** 수량 변경/삭제 기능 동작 확인
    
    - `btn-apply`, `btnDelete` 핸들러 보완
        
    - `cart:updated` 이벤트 트리거 유지
        

---

### Story 3. cart_proc.php API 확장

- **Task 3.1** `add_shop_cart` 프로시저 수정
    
    - IN: user_id, product_id, qty, option_hash
        
    - OUT: po_cart_id(seq_no), po_qty_after
        
    - ON DUPLICATE KEY UPDATE 시 alias 사용 (MySQL 8 deprecation 대응)
        
- **Task 3.2** cart_proc.php `add` 응답 스키마 확정
    
    - `{ ok: true, cart_id, qty_after }`
        
- **Task 3.3** update_shop_cart 프로시저 개선
    
    - UPDATE 후 `LAST_INSERT_ID()` 대신 입력 cart_id 반환
        
    - qty <= 0 시 삭제 처리
        

---

### Story 4. order_form.php 상품조회 프로시저 연동

- **Task 4.1** `shop_cart_selected_order_form` 프로시저 작성
    
    - IN: user_id, cart_ids_json
        
    - 결과셋:
        
        1. 회원/배송지 정보
            
        2. 아이템 목록 (cart_id, product_id, product_name, final_price, sale_price, dis_rate, quantity, row_sum …)
            
        3. 합계/배송비 (subtotal, shipping, discount, total)
            
- **Task 4.2** order_form.php 파싱 로직 구현
    
    - selected_cart_ids 배열 수신 → JSON 인코딩 → CALL
        
    - nextRowset()으로 3개 result set fetch
        
- **Task 4.3** product_data 수동 생성
    
    - `$products_info`에서 `{id, qty}` 배열 생성 → JSON 직렬화
        
    - product_data가 없을 경우 기본값 `"[]"` 방어 처리
        

---

### Story 5. DB 구조 및 뷰 개선

- **Task 5.1** shop_cart UNIQUE KEY (user_id, product_id, option_hash) 설정
    
- **Task 5.2** v_products_effective 뷰 정리
    
    - final_price 칼럼 제거 → 동적 계산식 사용
        
    - 할인율(discount_rate) 반영:
        
        `ROUND(p.sale_price * (1 - COALESCE(p.discount_rate, 0)), 0) AS final_price`
        
- **Task 5.3** 배송비 계산 로직 강화
    
    - grouped CTE: 정책/묶음 단위 그룹핑
        
    - shipping_calc: 정책별 배송비 합산
        
    - subtotal_calc: 전체 소계 계산
        
    - MySQL 5.7 호환 고려 시 → 임시테이블 방식으로 대체
        

---

# QA Checklist

-  바로구매 시 신규 row/기존 row qty 가산 처리 정상 작동
    
-  장바구니 선택구매: 선택/해제/수량변경 후 주문서 전달 값 확인
    
-  전체구매/선택구매 product_data JSON 구조 검증
    
-  배송비 계산 free_over 조건 임계값(예: 정확히 30,000원) 테스트
    
-  product_data 누락 시 기본값 `"[]"` 정상 동작