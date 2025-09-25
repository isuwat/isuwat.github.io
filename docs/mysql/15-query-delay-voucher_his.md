---
layout: default
title: "[MySQL] 쿼리지연-voucher 더보기" 
parent: MySQL
nav_order: 15
---




# 1) 지연 원인 & 패치

#### 인덱스(최소 세트)

* gpt 추천

```sql
-- 내가 개봉자 히스토리
CREATE INDEX ix_voucher_user_regdate_seq
ON voucher (user_id, regdate, seq_no);

-- 내가 등록자 히스토리
CREATE INDEX ix_voucher_restore_user_regdate_seq
ON voucher (restore_user_id, regdate, seq_no);

-- (선택) 상태별 최근순 조회 빈번하면
ALTER TABLE voucher ADD KEY ix_voucher_stat_regdate (stat, regdate);
```

* 실제 적용
```sql
-- 1. voucher index
CREATE INDEX ix_voucher_user_regdate_seq
ON voucher (user_id, regdate, seq_no);

-- 내가 등록자 히스토리
CREATE INDEX ix_voucher_restore_user_regdate_seq
ON voucher (restore_user_id, regdate, seq_no);
```

## 무엇이 느리게 하나요?

- **CASE 기반 WHERE**: `pi_stat` 값에 따라 `user_id`/`restore_user_id`/`serial_code`를 바꿔가며 비교하는 단일 CASE 문은 옵티마이저가 최적 인덱스를 고르기 어렵게 합니다. 특히 “전체” 분기의 `OR` 조합까지 섞여 있어 범위가 넓어져요.
    
- **집계 쿼리에서 불필요한 정렬**: `COUNT,SUM` 용 쿼리에 `ORDER BY regdate DESC`가 들어 있어 파일소트/정렬 버퍼를 추가로 탑니다(집계에 정렬은 필요 없음).
    
- **PIN 조회(`pi_stat=9`)에서 날짜 강제 확장**: `2024-01-01~NOW`로 기간을 넓혀 놓아 인덱스 선택을 헷갈리게 만듭니다. PIN은 보통 `serial_code = ?` 단건 탐색이면 충분합니다.
    

## “빠른 체감” 패치 (쿼리만 교체)

아래 블록만 교체해도 눈에 띄게 빨라집니다.

### A) 집계(카운트/합계) 블록 교체

속도 개선됨

```sql
-- [OLD] CASE + ORDER BY regdate DESC 제거 대상
-- SELECT COUNT(*), SUM(price) ... WHERE CASE ... AND regdate BETWEEN ... ORDER BY regdate DESC;

-- [NEW] 분기별 정적 WHERE + 전체는 UNION ALL
IF pi_stat = 1 THEN
  SELECT COUNT(*), IFNULL(SUM(price),0)
	...

ELSEIF pi_stat = 5 THEN
  SELECT COUNT(*), IFNULL(SUM(price),0)
	...

ELSEIF pi_stat = 9 THEN
  -- PIN 단건: 날짜 조건 제거(또는 매우 좁게), serial_code 고유 인덱스 사용
	...

ELSE
  -- 전체: 개봉/등록 두 갈래를 UNION ALL로 합산 → 각기 최적 인덱스 사용
  SELECT COUNT(*), IFNULL(SUM(price),0)
    INTO po_total_cnt, po_total_price
  FROM (
    SELECT price
	...
    UNION ALL
    SELECT price
	...
  ) x;
END IF;

```

근거: 기존 CASE 기반 WHERE/정렬/리밋 구성.

> **효과**: 단일 CASE → **분기별 정적 WHERE/UNION ALL** 로 바꾸면 각 분기에서 **딱 맞는 인덱스**를 집습니다. “전체”도 두 갈래를 따로 타서 합치기 때문에 훨씬 덜 느립니다.