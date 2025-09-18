---
layout: default
title: "[admin] 주문조회 요약"
parent: admin
nav_order: 6
---

# 주문 조회 요약


# Story 제목

**[관리자] 주문조회 개편: 대표주문 1행 + 상품명/전화 검색 강화**

## 간단 설명

대표주문 기준으로 결과를 1행씩 고정하고, 주문에 담긴 모든 상품명을 줄바꿈으로 표기. 상품명 LIKE 검색과 전화번호(회원/수령인) 검색을 지원하며, 주문취소/결제취소 상태도 조회/표시에 반영.

---

## Acceptance Criteria

- 대표주문 N건 조회 시 결과도 **정확히 N행**.
    
- 각 행에 주문상품 전체가 **줄바꿈**으로 표시(`상품명 x수량`).
    
- **상품명 LIKE 검색**: 부분 일치로 해당 상품이 포함된 주문만 필터.
    
- **전화번호 검색**: 회원/수령인 전화 모두 대상, 하이픈 유무 상관없이 검색.
    
- **주문상태**: 주문취소/결제취소(환불) 포함 표기 및 필터 가능.
    
- 성능: 1만 건 범위 데이터에서 필터/정렬/페이징 모두 **1초 내** 응답(개발 환경 기준).
    

---

## Tasks

### DB / Query

-  대표주문 1행 쿼리로 전환
    
    - `shop_order_product` 는 직접 조인하지 말고 **집계 서브쿼리(agg)** 로 1행 조인.
        
    - `GROUP_CONCAT(op.product_name … SEPARATOR 0x0A)` 로 줄바꿈 문자열 `product_lines` 생성.
        
    - `SUBSTRING_INDEX(GROUP_CONCAT …,'||',1)` 로 첫 상품명 추출 → `product_summary` (“첫 상품 외 N건”) 표기.
        
-  **상품명 LIKE 검색** 추가
    
    - 메인 쿼리 WHERE에 `EXISTS (SELECT 1 FROM shop_order_product op2 WHERE op2.order_seq_no = ods.seq_no AND op2.product_name LIKE CONCAT('%', :kw_esc, '%') ESCAPE '\\')`
        
    - LIKE 특수문자(`% _ \`) 이스케이프 처리.
        
-  **전화번호 검색(회원/수령인)**
    
    - 숫자만 비교: `REPLACE(REPLACE(REPLACE(phone,'-',''),' ',''),'+82','0')` 형태로 정규화 후 `LIKE`.
        
    - 대상 컬럼: `ods.user_phone`, `ods.receiver_phone`(존재 시).
        
-  **PG 로그 최신 1행** 서브쿼리 조인(`MAX(id)`).
    
-  **택배사명** 조인: `shop_couriers` 1:1.
    
-  **상태 매핑·필터**
    
    - 표시: `pending→결제전`, `paid→결제완료`, `preparing→상품준비중`, `shipping→배송중`, `delivered→배송완료`, `complete→구매확정`, `cancelled→주문취소`, `refunded→결제취소/환불완료`.
        
    - 필터: `cancelled`, `refunded` 포함 동작 확인.
        
-  세션 한도 조정(대량 주문 대비)
    
    - `SET SESSION group_concat_max_len = 65535;`
        
-  인덱스 정비
    
    - `shop_order_product (order_seq_no, product_name)`
        
    - `shop_orders (order_status, order_date)` / `(order_date)`
        
    - 전화 검색 최적화가 필요하면 `user_phone`/`receiver_phone`에 **generated column(숫자만)** + 인덱스 고려.
        

### API / 백엔드

-  관리자 조회 API 파라미터 확장
    
    - `product_name`(부분검색), `phone`(하이픈 무시), 기존 기간/상태/정렬/페이지 유지.
        
-  입력값 이스케이프 & 바인딩
    
    - LIKE용 이스케이프(`\%`, `\_`, `\\`), 전화번호 숫자만 추출 후 바인딩.
        
-  응답 필드
    
    - `product_lines`(줄바꿈 포함), `product_summary`, `pay_type(표기)`, `carrier_name`, `result_msg`.
        

### UI

-  **상품명 줄바꿈 표시**
    
    - `nl2br(htmlspecialchars($row['product_lines']))` 로 안전 출력.
        
-  검색 폼
    
    - 상품명 입력(LIKE), 전화번호 입력(하이픈 무관), 상태/기간/정렬/페이지 유지.
        
-  열 구성/말줄임
    
    - 상품요약 칼럼은 라인 높이/최대 높이 지정, 필요 시 “더보기” 처리.
        

### QA / 테스트

-  케이스: 상품 1개/다개, 동일 주문 내 다수 상품명, 긴 상품명.
    
-  LIKE 특수문자 포함(`%`,`_`,`\`) 검색 동작.
    
-  전화번호: `010-1234-5678`, `01012345678`, 국가코드 변형(`+82`) 등 검색.
    
-  상태별 필터(특히 `cancelled`, `refunded`) 및 표기 정확성.
    
-  성능: 상품 50+ 포함 주문 다수 시 응답 시간 측정.
    
-  XSS/SQLi: 출력 이스케이프·바인딩 정상.
    

### 문서 / 운영

-  쿼리/인덱스 변경 내역 위키 반영.
    
-  롤백 전략(구 쿼리 스위치) 기록.
    
-  운영 배포 체크리스트(세션 변수 설정 포함).
    

---

## 참고 구현 포인트(요약 스니펫)

- 상품 줄바꿈 문자열:
```sql
GROUP_CONCAT(
  CONCAT(op.product_name, ' x', op.quantity, '개')
  ORDER BY op.reg_date ASC
  SEPARATOR 0x0A
) AS product_lines

```

상품명 LIKE EXISTS:
```sql
AND (
  :product_name IS NULL OR :product_name = '' OR EXISTS (
    SELECT 1
    FROM shop_order_product op2
    WHERE op2.order_seq_no = ods.seq_no
      AND op2.product_name LIKE CONCAT('%', :kw_esc, '%') ESCAPE '\\'
  )
)

```