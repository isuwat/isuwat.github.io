---
layout: default
title: "[admin] 주문상태 ENUM 필드 삭제"
parent: admin
nav_order: 4
---

# [DB] 주문상태 ENUM 필드 삭제 및 TINYINT 전환

## 배경/목표

- `shop_orders.order_status` 타입을 **ENUM → TINYINT**로 전환하고, 전 시스템(프로시저/페이지)에서 상태 비교/표시 로직을 **숫자 코드 기준**으로 일원화한다.
    
- 취소 상태를 **7=관리자취소**, **9=사용자취소**로 분리 유지.
    

## 상태 코드 매핑 (최종)

```ini
1=pending, 2=paid, ...

```


---

## 작업 범위 (Scope)

### 1) DB 스키마/데이터 마이그레이션

- `shop_orders.order_status` 컬럼 ENUM → **TINYINT UNSIGNED NOT NULL**로 변경 (코멘트에 코드표 기입).
    
- (데이터 이관 완료됨) 문자열 → 코드 변환 검증:
    
    - 예: `UPDATE shop_orders SET order_status = CASE order_status ... END;` 형태로 1:1 매핑.
        
- 인덱스/뷰/트리거 영향도 없음(상태 비교만 변경).
    

### 2) 프로시저 (App)

- `select_shop_my_orderstate` **v v**
    
    - 상태 카운트/분기 숫자 기준(`IN (2,3,4)` 등)으로 교체.
        
- `select_shop_order_list` **v v**
    
- `select_shop_order_detail` **v v**
    
- `shop_order_insert` **v** (`pending` → **1**) **v**
    
- `shop_order_update` 파라미터 `pi_order_stat` → **TINYINT** **v v**
    
- `pl_shop_order_insert` **v** (`pending` → **1**) **v**
    
- `pl_shop_order_update` **x** (미적용/보류)
    
- `shop_pl_callback` **v** (PHP 수정 없음, 프로시저 내부 숫자 기준으로 **vv**)
    

### 3) 관리자 프로시저/UI

- `select_adm_shop_orders_list`
    
    - 필터 바인딩(셀렉트박스) 숫자 코드 적용 **vv**
        
- `select_adm_shop_order_detail`
    
    - `CASE` 매핑 숫자 기준으로 교체 **vv**
        
- `update_adm_shop_order_dev` (MD 상태변경)
    
    - `p_value` 문자열/숫자 모두 허용 → 내부에서 TINYINT 매핑, 전 분기 숫자 비교로 통일 **vvv**
        

### 4) PHP 페이지 (관리자/앱)

- 앱
    
    - `myshop/orderlist.php` line:53 → 숫자 상태 코드 사용 **v**
        
    - `cash_pay_req.php` line:205
        
        - `$order_stat = $res['rsltCd'] == '0000' ? 2 : 12;` (성공=2/실패=12) **확인 완료**
            
- 관리자
    
    - `orders/order_list`
        
        - line 447: 상태 셀렉트박스 **vv**
            
        - line 634: MD 주문확인 버튼 조건 **vv**
            
    - `orders/order_detail`
        
        - line 447: 상태 셀렉트박스 **vv**
            
        - line 634: MD 주문확인 버튼 조건 **vv**
            

> 버튼 활성 조건 예:
> 
> - MD 주문확인: `order_status === 2`
>     
> - 송장입력: `order_status ∈ {3,4}`
>     
> - 배송완료 처리: `4 → 5`, `5/6 멱등`
>     
> - 배송 이후 취소/환불 차단: `v_status ∈ {4,5,6} && v_new_status ∈ {7,8}`
>