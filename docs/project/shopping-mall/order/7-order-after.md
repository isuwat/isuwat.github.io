---
layout: default
title: "[order] 결제 프로시저-결제 후"
parent: order
nav_order: 7
---


# shop_order_update (결제 후)
목적: **PG 콜백/리다이렉트 결과**를 받아 주문을 확정/실패로 전이.

## 입력/출력(예시)

- IN: `pi_order_no` 또는 `pi_mTrdNo`(PG 거래키), `pi_pay_status`, `pi_approval_no`, `pi_paid_amount`, `pi_paid_at`, `pi_signature` …
    
- OUT: `po_rc`, `po_msg`
    

## 처리 단계

1. **검증 전 빠른 차단**
    
    - **멱등성**: 이미 `order_status IN ('paid','canceled','failed')`면 **재처리 금지**(현재 상태 반환)
        
2. **START TRANSACTION**
    
3. **주문 행 잠금**: `SELECT … FROM shop_orders WHERE order_no=? FOR UPDATE`
    
4. **무결성/보안 검증**
    
    - 금액: `pi_paid_amount == shop_orders.total_payment` (±1원 허용 여부 정책)
        
    - 서명/무결성: `pi_signature`(PG 제공 값) 검증
        
    - 중복 승인번호: `approval_no` 유니크 체크(중복 결제 방지)
        
5. **상태 전이**
    
    - 성공: `order_status='paid'`, `approval_msg`, `approval_no`, `paid_at` 업데이트
        
    - 실패/취소: `order_status='failed'` 또는 `canceled` + 사유 저장
        
6. **부가 처리**
    
    - 사용자 캐시 차감/정산 필요 시 반영(현재는 선차감/후차감 정책에 맞춤)
        
    - 영수증/세금계산서 발행 트리거
        
    - 재고가 insert에서 이미 확정되었다면 OK, 아니라면 이 시점 차감
        
7. **결제 로그 테이블 insert** (결과/코드/전문 원문 일부)
    
8. **COMMIT**
    
9. OUT: `po_rc=0`, `po_msg='OK'` (또는 오류코드/메시지)
    

## 주의

- **리트라이 안전**: 동일 `pi_mTrdNo`/`approval_no` 재전달 시 **멱등 처리**.
    
- **시그널/RESIGNAL**: 예외 발생 시 롤백 후 호출부에서 try/catch.