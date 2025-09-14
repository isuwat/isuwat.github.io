---
layout: default
title: "[order] 결제 프로시저-결제 전"
parent: order
nav_order: 6
---


# shop_order_insert (결제 전)


목적: **주문 생성·검증·장바구니 정리**까지를 한 트랜잭션으로 처리해 “결제요청 가능한 주문”을 만든다.

## 입력/출력(예시)

- IN: `pi_user_id`, `pi_trdAmt(표시총액)`, `pi_selected_cart_ids(JSON)`, 배송지 정보…
    
- OUT: `po_order_no`, `po_total_order_price(서버재계산)`, `po_rc`, `po_msg`
    

## 처리 단계

1. **START TRANSACTION**
    
2. **장바구니 행 잠금** `SELECT … FOR UPDATE` (선택된 seq_no + user_id)
    
3. **합계 재계산** `CALL calc_cart_totals(...)`
    
    - `v_rc != 0` → SIGNAL
        
    - `ABS(pi_trdAmt - v_total) > 1` → `PRICE_CHANGED`
        
4. **주문 헤더 생성**: `order_status='pending'`(또는 `ready_for_payment`)
    
    - `order_number=pi_mTrdNo`(PG 트랜잭션키) 저장
        
    - `total_payment = v_total`(서버 기준)
        
5. **주문 상세 INSERT**: 장바구니 기반(`final_price × qty` 스냅샷)
    
6. **장바구니 삭제**: 선택 항목만
    
7. **로그/감사 기록** (요청 생성 로그)
    
8. **COMMIT**
    
9. OUT: `po_order_no`, `po_total_order_price=v_total`, `po_rc=0`, `po_msg='OK'`
    

## 주의

- insert 단계에서는 **결제 확정/승인값(승인번호 등)** 저장하지 않음.
    
- 멱등성 토큰(예: `order_number`+`user_id`) 유니크로 **중복 생성 방지**.