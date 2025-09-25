---
layout: default
title: "[admin] 주문상태 변경"
parent: admin
nav_order: 7
---



## 목적

- MD 업무 플로우를 주문 단위로 처리:
    
    - **MD 주문확인 → 송장 입력(배송중) → 배송완료**
        
- 외부(도매몰) 주문번호를 **주문 헤더에 저장/표시**.
    
- 모든 변경은 **shop_order_logs**에 자동 기록.
    

---

## 사용자 흐름(요약)

1. **MD 주문확인**: `paid → preparing` (멱등)
    
2. **MD 송장입력**: 송장/택배사 저장 + `paid|preparing → shipping` 승격
    
    - **어드민 송장입력(일반)**: 송장/택배사 저장만(상태 전이 없음)
        
3. **MD 배송완료**: `shipping → delivered` (멱등)
    
4. **도매몰 주문번호 입력**: 헤더 필드에 저장(상태 전이 없음)
    

---

## DB/스키마

- 사용 컬럼(기존/신규)
    
    - `shop_orders.tracking_number` (송장)
        
    - `shop_orders.courier_id` (택배사)
        
    - **신규** `shop_orders.vendor_order_no` (도매몰 주문번호)

```sql
ALTER TABLE shop_orders
  ADD COLUMN vendor_order_no VARCHAR(64) NULL COMMENT '도매몰 주문번호' AFTER tracking_number,
  ADD INDEX ix_vendor_order_no (vendor_order_no);

```
        
        
- 로그    
    - `shop_order_logs(order_seq_no, log_date, log_message)`에 자동 기록
        

---

## 프로시저: `update_adm_shop_order` 주요 액션

- 공통 입력:  
    `p_order_seq_no, p_action, p_value, p_pd_seq_no(NULL), p_courier_id(NULL), OUT po_result_code, OUT po_err_msg`
    
- 상태 전이/검증 로직(핵심만)
    
    - `md_confirm`: `paid → preparing` (그 외 상태면 멱등/거부)
        
    - `md_invoice`: 송장/택배사 저장 + `paid|preparing → shipping` 승격 (이미 `shipping`이면 멱등)
        
    - `tracking`(일반 어드민): 송장/택배사 저장만(전이 없음), 허용 상태 `paid|preparing|shipping`
        
    - `md_delivered`: `shipping → delivered` (그 외 상태면 멱등/거부)
        
    - `set_vendor_order`: `vendor_order_no` 저장만(전이 없음), 허용 상태 `paid|preparing|shipping`
        
- 구현 포인트
    
    - 현재 상태/기존값을 **잠금 후 로컬 변수**로 취득:

```sql
SELECT order_status, tracking_number, vendor_order_no   
INTO v_status, v_old_tracking, v_old_vendor_no 
FROM shop_orders 
WHERE seq_no = p_order_seq_no 
FOR UPDATE;
```

        
    - 로그 메시지에서 **컬럼 직접 비교 금지** → `v_old_*` 변수 사용
        
    - 전이 불가 상태(예: 배송 시작 후 취소/환불)는 `SIGNAL`로 차단
        

---

## API 라우팅(`/admin/api/md_order_proc.php`)

- action 매핑

```sql
$actionMap = [
  'md-confirm'   => 'md_confirm',
  'md-invoice'   => 'md_invoice',
  'md-delivered' => 'md_delivered',
  'tracking'     => 'tracking',
  'status'       => 'status',
  'set-vendor'   => 'set_vendor_order',
];

```
    
    
- 파라미터
    
    - 공통: `order_id`
        
    - `md-invoice`/`tracking`: `invoice`, `courier_id`
        
    - `set-vendor`: `vendor_no`
        
- CALL 예시

```php
CALL update_adm_shop_order(:order_seq_no, :action, :value, :pd_seq_no, :courier_id, @rc, @msg);
SELECT @rc rc, @msg msg;

```
    
    
---

## 조회(대표주문 1행) 반영

- SELECT에 표시 필드 포함:
    
    - `ods.tracking_number AS order_tracking_number`
        
    - `ods.vendor_order_no AS vendor_order_no`
        
    - 상품 줄줄이: `agg.product_lines`, 링크용 `agg.product_ids`
        
    - (필요 시) 최신 PG 로그 1건, 택배사명 조인
        

---

## UI/JS

- **대표주문 1행**에만 주문 단위 액션 노출
    
- 표시(줄바꿈/좌측정렬)
    
    `<div>송장번호: <?= h($row['order_tracking_number'] ?? '-') ?></div> <div>도매몰주문번호: <?= h($row['vendor_order_no'] ?? '-') ?></div>`
    
- 입력
    
    - 도매몰 주문번호: `<input.vendor-no> + <button data-act="set-vendor">`
        
    - 송장/택배사: `<select.carrier> + <input.invoice> + <button data-act="tracking"> / <button data-act="md-invoice">`
        
- 공용 핸들러(발췌)

```js
$(document).on('click', '.md-actions [data-act]', async function(){
  const $btn = $(this), act = $btn.data('act'), oid = $btn.data('order-id');
  const payload = { action: act, order_id: oid };

  if (act === 'set-vendor') {
    payload.vendor_no = ($btn.closest('.md-actions').find('input.vendor-no').val()||'').trim();
    if (!payload.vendor_no) return alert('도매몰 주문번호를 입력하세요.');
  }
  if (act === 'tracking' || act === 'md-invoice') {
    const $w = $btn.closest('.md-actions');
    payload.courier_id = parseInt($w.find('select.carrier').val(),10) || null;
    payload.invoice    = ($w.find('input.invoice').val()||'').trim();
    if (!payload.invoice) return alert('송장번호를 입력하세요.');
  }

  try {
    const res = await $.post('/admin/api/md_order_proc.php', payload);
    if (!res?.ok) throw new Error(res?.msg || '처리 실패');
    location.reload();
  } catch(e){ alert(e.message); }
});

```
    
    

---

## 수용 기준(AC)

- 허용된 전이만 성공:  
    `paid → preparing → shipping → delivered`
    
- `md_invoice`는 전이(승격) 포함, `tracking`은 전이 없음.
    
- `set_vendor_order`는 값 저장만(전이 없음), 허용 상태 이외엔 차단.
    
- 모든 액션은 `shop_order_logs`에 메시지 기록.
    
- 목록/상세에 송장/도매몰 주문번호 표기, 빈 값은 `-` 또는 비표시.
    

---

## 테스트 체크리스트

- 상태별 성공/차단 케이스:
    
    - `md_confirm`: `paid`만 성공, 나머지 멱등/에러 메시지 확인
        
    - `md_invoice`: `paid|preparing`→`shipping` 승격, `shipping`에서 멱등 갱신
        
    - `tracking`: `paid|preparing|shipping`에서만 저장, 완료/취소 후 차단
        
    - `set_vendor_order`: `paid|preparing|shipping`에서만 입력/변경
        
- 입력 검증: 빈 값/공백, courier_id 유효성
    
- 로그 내용: “입력” vs “변경 (이전 → 신규)”
    
- UI: 상품 줄줄이 링크, 좌측정렬/줄바꿈 표시
    
- 회귀: 검색/페이징/정렬 영향 없음
    

---

## 배포/운영 메모

- **순서**: DDL(컬럼/인덱스) → 프로시저 → API → UI/JS
    
- 롤백: 코드 롤백 우선, 컬럼 드롭은 보류 가능
    
- 성능: 대표주문 1행 쿼리 사용, `GROUP_CONCAT` 길이 필요 시 `group_concat_max_len` 확대