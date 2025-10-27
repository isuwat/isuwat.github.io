---
layout: default
title: "[MySQL] 쿼리지연-마이페이지-충전내역" 
parent: MySQL
nav_order: 28
---



### 🎯 목적

기존 `UNION ALL` 방식에서 발생하던 전역 정렬(filesort)과 파생 테이블 머티리얼라이즈 비용을 제거하고,  
CTE(Common Table Expression) + Late Materialization 구조로 변경하여 **페이지당 응답 속도를 대폭 개선**한다.  
(테스트 기준: 약 0.9s → 0.06s)

### 📋 주요 개선 내용

- 기존 `UNION ALL` 구조 제거 → CTE(`WITH top_keys AS (...)`) 적용
    
- orders / voucher_orders 키만 먼저 정렬 후 LIMIT (Top-N key scan)
    
- 본문은 seq_no 조인 방식으로 최소 접근
    
- COUNT 쿼리 최적화: UNION COUNT → 테이블별 COUNT 합산
    
- 인덱스 추가: `(user_id, order_stat, order_date)`
    
- 페이징 계산 로직 보강 (v_list 통일, LEAVE 처리 유지)

#### INDEX 추가

```sql
/*
 * [운영DB 인덱스 추가]
 * ----------------------------------------------------------
 * 목적 :
 *   - 마이페이지 내역 조회(select_myp_order_list) 프로시저의
 *     전역 정렬 속도 개선 (filesort 제거, 인덱스 정렬 활용)
 *   - user_id + order_stat + order_date 로
 *     WHERE + ORDER BY 조합을 인덱스에서 직접 처리 가능
 *
 * 영향 :
 *   - 읽기(SELECT) 성능 향상, 쓰기(INSERT/UPDATE) 약간 증가 가능
 *   - 신규 인덱스 크기 : 약 (8~12byte * row_count)
 *   - ONLINE DDL 옵션 사용 시 서비스 영향 최소
 *
 * 검증 :
 *   - 실행 전: SHOW INDEX FROM orders;
 *   - 실행 후: EXPLAIN ANALYZE 로 key=idx_orders_user_stat_date 확인
 * ----------------------------------------------------------
 */
ALTER TABLE orders
  ADD INDEX idx_orders_user_stat_date (user_id, order_stat, order_date);

-- 삭제(롤백)용
-- ALTER TABLE orders DROP INDEX idx_orders_user_stat_date;


ALTER TABLE voucher_orders
  ADD INDEX idx_vorders_user_stat_date (user_id, order_stat, order_date);
```

#### CTE 방식으로 top_key 우선조회 후 본문 조회

## 🔹 CTE 방식 핵심 개념

> **CTE(Common Table Expression)** = 쿼리의 일시적인 결과 집합을 만들어  
> 메인 쿼리에서 재사용하는 구조.
> 
> 이번 구조에선 이 CTE가 **“키 전용 정렬 테이블”** 역할을 합니다.

---

## 🔹 기존 방식 (문제점)

기존 `UNION ALL` 구조에서는:

1. `orders`, `voucher_orders`에서 모든 행을 조회
    
2. 두 결과를 **전부 합친 뒤**
    
3. `ORDER BY order_date DESC`로 전역 정렬
    
4. 마지막에 `LIMIT`으로 필요한 행만 추림
    

➡️ 결과적으로

- 불필요한 수만 건 정렬
    
- `filesort` 발생
    
- 임시 테이블/CPU 사용량 폭증
    

---

## 🔹 CTE 방식 (개선 아이디어)

새 쿼리는 **“Late Materialization (지연 조회)”** 구조로 되어 있습니다.

즉,

> **1단계: 키(정렬 기준)만 가볍게 정렬하고 LIMIT**  
> **2단계: 그 키로 실제 행 데이터를 조인해서 가져온다**

---

## 🔹 실행 순서 (이해 포인트)

### ① CTE – 키 전용 정렬

```sql
WITH top_keys AS (
  SELECT 'O' AS src, o.seq_no, o.order_date
  FROM orders o
  WHERE o.user_id = ?
    AND o.order_stat = '1'
  UNION ALL
  SELECT 'V' AS src, v.seq_no, v.order_date
  FROM voucher_orders v
  WHERE v.user_id = ?
    AND v.order_stat IN ('1','5')
  ORDER BY order_date DESC, seq_no DESC
  LIMIT ?, ?
)
```

- 이 단계에서 **정렬 기준이 되는 컬럼만(select seq_no, order_date)** 모음
    
- `ORDER BY … LIMIT`으로 **필요한 행의 키만** 남김  
    (예: 1000개 중 상위 50개)
    
- 메모리에 작은 키 테이블(`top_keys`)만 존재

② 본문 조인 – 실제 데이터 가져오기

```sql
SELECT o.*
FROM top_keys tk
JOIN orders o ON tk.src='O' AND o.seq_no=tk.seq_no
UNION ALL
SELECT v.*
FROM top_keys tk
JOIN voucher_orders v ON tk.src='V' AND v.seq_no=tk.seq_no;
```
- 이제 정렬된 **키 집합(top_keys)** 과 실제 테이블을 `seq_no`로 조인
    
- 불필요한 행 접근 없이, **정확히 필요한 데이터만** 가져옴
    
- 최종 결과에서 `ORDER BY order_date_s DESC, seq_no DESC`로 순서 유지
## 🔹 장점 요약

|항목|기존 UNION 방식|CTE (Late Materialization) 방식|
|---|---|---|
|정렬 대상|전체 행|키(필요한 행만)|
|정렬 비용|높음 (filesort)|낮음 (작은 인메모리 정렬)|
|메모리 사용|큼 (모든 열 포함)|작음 (3개 컬럼만)|
|결과 정확성|동일|동일|
|실행 속도|약 0.8~1.0s|**0.05~0.06s** (테스트 기준)|

---

## 🔹 한 줄 요약

> **“CTE(top_keys)”는  
> ‘정렬 기준이 되는 키만 먼저 빠르게 뽑은 뒤,  
> 그 키로 필요한 데이터만 정확히 가져오는’ 방식입니다.**
> 
> 덕분에 정렬/스캔 비용이 거의 사라지고,  
> `UNION ALL` + 전역 정렬보다 훨씬 빠릅니다.