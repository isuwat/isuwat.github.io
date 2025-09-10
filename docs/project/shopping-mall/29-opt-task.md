---
layout: default
title: "[option] 옵션 작업(Task)"
parent: shop
nav_order: 29
---

# 옵션 작업 태스크

## 목표

- 상세페이지에서 **메인 옵션(필수)만**으로 구매 가능하도록 구축
    
- 추후 애드온을 붙일 수 있도록 **호출·데이터 구조는 확장성 유지**
    

## 수용 기준(AC)

1. 상세에 **단일 `<select>`** 표시, 첫 항목 `- [필수] 옵션을 선택해 주세요 -`, 품절/비활성은 `<option disabled>` + “(품절)” 표기.
    
2. 옵션 선택 시 **AJAX**로 `{variant_id, price, stock_qty, img_url?}` 갱신, 재고 0이면 구매 버튼 비활성.
    
3. 장바구니 담기 시 서버에서 **product-variant 매칭/활성/재고 검증** 후 저장(스냅샷 포함).
    
4. 주문 성공시에만 재고 차감(트랜잭션), 실패 시 롤백.
    
5. 코드 구조는 이후 **애드온 추가 액션**(별도 API) 붙여도 충돌 없이 확장 가능.


## 산출물(간단)

- DB/프로시저
    
    - `product_variant`: `variant_id, product_id, sku_code, option_label, price, stock_qty, is_active, img_url`
        
    - `sp_get_product_variants(pi_product_id)`
        
    - `sp_get_variant_by_sku(pi_sku_code)`
        
    - `sp_cart_add(pi_user_id, pi_product_id, pi_variant_id, pi_qty, OUT po_rc, OUT po_msg)`
        
    - (주문) `sp_order_create_from_cart(...)` — 기존 사용 시 그대로
        
- API (`api/cart_proc.php`)
    
    - `GET action=variant_by_sku&sku=...`
        
    - `POST action=add` (메인만 담기)
        
- 프런트/뷰
    
    - `detail.php`: 단일 셀렉트 + AJAX + 히든 `variant_id` + “장바구니 담기”
        
    - `basket.php`: 기존 유지(카운트 갱신 이벤트만 사용)
        

---

## 태스크(실행 순서)

### 1) DB/프로시저

- **T1 스키마 확정**: `product_variant` 인덱스(`product_id`, `sku_code`)
    
- **T2 SP 작성**
    
    - `sp_get_product_variants(pi_product_id)` → 옵션 목록
        
    - `sp_get_variant_by_sku(pi_sku_code)` → 선택 SKU 상세
        
    - `sp_cart_add(...)` → 검증·스냅샷·rc/msg
        

### 2) API

- **T3 GET `action=variant_by_sku`**
    
    - IN: `sku` → OUT: `{variant_id, price, stock_qty, img_url?}`
        
    - 200/400/5xx 정리
        
- **T4 POST `action=add`**
    
    - IN: `product_id, variant_id, qty` → OUT: `{rc, msg}`
        
    - 미로그인 401, 재고부족 409 등 표준화
        

### 3) 상세페이지(UI/JS)

- **T5 렌더링** (`detail.php`)
    
    - `<select>`: 안내/구분선/`<optgroup>`/품절 비활성
        
    - `<input type="hidden" id="variant_id">`
        
- **T6 선택 핸들러(JS)**
    
    - `change` → `variant_by_sku` 호출 → 가격/재고/이미지 갱신
        
    - `stock_qty==0` 시 구매버튼 비활성
        
    - 실패 시 리셋/알림
        
- **T7 장바구니 제출(JS)**
    
    - `action=add`로 폼/AJAX 전송 → 성공 시 `cart:updated` 이벤트
        

### 4) QA/문서

- **T8 테스트**
    
    - 옵션 미선택 차단, 품절 비활성, 잘못된 SKU, 네트워크 실패
        
- **T9 문서**
    
    - 응답 포맷/에러코드, 셀렉트 라벨 규칙(다옵션일 때 “색상 / 사이즈”)

