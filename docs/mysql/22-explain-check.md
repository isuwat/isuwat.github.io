---
layout: default
title: "[MySQL] EXPLAIN 검증" 
parent: MySQL
nav_order: 22
---



**지금 만든 인덱스 3종**(regdate, user_id, restore_user_id 축) 기준으로,  
“키만 정렬 → 본문 JOIN(두 단계 페치)” 구조에 맞춘 **EXPLAIN 검증 스크립트**를 드릴게요.  
아래 쿼리들을 그대로 실행해서 `type / key / rows / Extra`를 확인하면 됩니다.

### 공통(예시 바인딩)

```sql
-- 기간(반열림 구간)
SET @start  := '2025-07-01';
SET @endNxt := '2025-10-02';

-- 페이징
SET @offset := 0;
SET @limit  := 50;

-- 필터 샘플
SET @user_id         := 'user_123';
SET @restore_user_id := 'restorer_9';
SET @stat            := 2;

```

### 1) 전역 목록(필터 없음) — 서브쿼리: “키만 정렬”

```sql
EXPLAIN
SELECT s.seq_no
FROM voucher s USE INDEX (ix_voucher_regdate_seq)
WHERE s.regdate >= @start AND s.regdate < @endNxt
ORDER BY s.regdate DESC, s.seq_no DESC
LIMIT @offset, @limit;
```
**OK 기대값**

- `type=range`
    
- `key=ix_voucher_regdate_seq`
    
- `Extra`에 `Using filesort`/`Using temporary` 없음

### 2) 사용자 필터 목록 — 서브쿼리

```sql
EXPLAIN
SELECT s.seq_no
FROM voucher s USE INDEX (ix_voucher_user_regdate_seq)  -- 먼저 USE INDEX 없이도 테스트해보세요
WHERE s.user_id = @user_id
  AND s.regdate >= @start AND s.regdate < @endNxt
ORDER BY s.regdate DESC, s.seq_no DESC
LIMIT @offset, @limit;
```

**OK**: `key=ix_voucher_user_regdate_seq`, `type=range` 또는 `ref`, `filesort` 없음

### 3) 복구자 필터 목록 — 서브쿼리

```sql
EXPLAIN
SELECT s.seq_no
FROM voucher s USE INDEX (ix_voucher_restore_user_regdate_seq)  -- 먼저 생략 후 확인
WHERE s.restore_user_id = @restore_user_id
  AND s.regdate >= @start AND s.regdate < @endNxt
ORDER BY s.regdate DESC, s.seq_no DESC
LIMIT @offset, @limit;
```

**OK**: `key=ix_voucher_restore_user_regdate_seq`, `filesort` 없음

# 4) stat=2 비교 — 서브쿼리

> 옵티마이저가 엉뚱한 인덱스를 고르면 느려질 수 있어 **regdate 인덱스를 유도**합니다.

① (힌트 없음)

```sql
EXPLAIN
SELECT s.seq_no
FROM voucher s
WHERE s.regdate >= @start AND s.regdate < @endNxt
  AND s.stat = @stat
ORDER BY s.regdate DESC, s.seq_no DESC
LIMIT @offset, @limit;
```

② (정렬 인덱스 유도)

```sql
EXPLAIN
SELECT s.seq_no
FROM voucher s USE INDEX (ix_voucher_regdate_seq)
WHERE s.regdate >= @start AND s.regdate < @endNxt
  AND s.stat = @stat
ORDER BY s.regdate DESC, s.seq_no DESC
LIMIT @offset, @limit;

```

### 5) 전체 카운트 + 합계(집계)

```sql
EXPLAIN
SELECT COUNT(*), SUM(price)
FROM voucher
WHERE regdate >= @start AND regdate < @endNxt
-- 선택적으로 동일 필터 부여
-- AND user_id = @user_id
-- AND stat = @stat
;

```

**OK**: `key=ix_voucher_regdate_seq`(있으면 사용), `type=range`  
(집계를 자주 치면 `(regdate, price)` 커버링 인덱스도 고려)

# 6) 바깥 쿼리(본문 JOIN) — 두 단계 페치 전체

> 서브쿼리에서 `rows=50` 내외만 나오고, 바깥 조인은 `eq_ref`로 붙는지 확인

```sql
EXPLAIN
SELECT v.*
FROM (
  SELECT s.seq_no
  FROM voucher s USE INDEX (ix_voucher_regdate_seq)
  WHERE s.regdate >= @start AND s.regdate < @endNxt
  ORDER BY s.regdate DESC, s.seq_no DESC
  LIMIT @offset, @limit
) t
JOIN voucher v ON v.seq_no = t.seq_no
ORDER BY v.regdate DESC, v.seq_no DESC;

```

**OK(대표 패턴)**

- 파생(derived) 단계: `type=range`, `key=ix_voucher_regdate_seq`, `rows≈@limit`, `filesort 없음`
    
- 바깥 `JOIN voucher v`: `type=eq_ref`, `key=PRIMARY`, `rows=1`

# 7) EXPLAIN ANALYZE(가능하면)

MySQL 8.0.18+ 에서 **실제 시간/행 수**까지 봅니다.

```sql
EXPLAIN ANALYZE
SELECT s.seq_no
FROM voucher s USE INDEX (ix_voucher_regdate_seq)
WHERE s.regdate >= @start AND s.regdate < @endNxt
ORDER BY s.regdate DESC, s.seq_no DESC
LIMIT @offset, @limit;

```

“`rows examined`가 LIMIT 근처로 작게”, “`sort` 관련 단계가 없음”이 이상적

## 빠른 해석 가이드

- **좋음**: `type=range/ref/eq_ref`, `key=...regdate_seq`/`...user_regdate_seq`, `rows` 적음, `Extra`에 `Using filesort`/`Using temporary` 없음
    
- **나쁨**: `type=ALL`, `key=NULL` 또는 엉뚱한 인덱스, `Using filesort` 있음 →  
    → 서브쿼리에 `USE INDEX (ix_voucher_regdate_seq)` 추가 또는 인덱스 보강
    

## 흔한 이슈 및 처방

- `ORDER BY`가 인덱스랑 안 맞음 → 정렬 키는 `regdate, seq_no` 고정
    
- OR/`1=1` 남아있음 → 동적 WHERE로 “있을 때만 등식” 추가
    
- `LIKE '%...%'` 등 비-SARG 조건 → 가능한 등식/전위매칭으로

### 6) 결과 검증

지금 EXPLAIN이 말해주는 것

```sql
2  DERIVED  s   range  ix_voucher_regdate_seq  ...  rows=459869  Using where; Backward index scan; Using index
1  PRIMARY  v   eq_ref PRIMARY                 ...               (t.seq_no = v.seq_no)
1  PRIMARY  <derived2> ALL                                          Using temporary; Using filesort

```

- **id=2 (서브쿼리)**: `(regdate, seq_no)` 인덱스를 **역방향(range) 스캔**하고 있고, `Using index`(커버링)까지 떠서 아주 좋아요. 우리가 의도한 “키만 정렬 + LIMIT” 플랜이 제대로 탔습니다.
    
- **id=1 (바깥)**: 최상위 `ORDER BY` 때문에 **Using temporary; Using filesort**가 찍입니다. 하지만 정렬 대상은 **딱 50행**(LIMIT 결과)이라 이 비용은 무시해도 됩니다.
    

> 요약: 이 플랜은 **정상/이상적**에 가깝습니다. 느림의 원인이 아니에요.

## 지금 플랜에서 꼭 볼 포인트

- 서브쿼리 `id=2`가 `key=ix_voucher_regdate_seq`, `Using filesort` **없음** → 이게 성능의 99%를 좌우합니다. 이미 OK.
    
- 바깥 `Using temporary; Using filesort`는 **50행 정렬**이므로 체감 거의 0.
    

---

## 체크리스트 (마무리 점검)

- `WHERE`는 반열림: `regdate >= :start AND regdate < :end_next` 유지 ✅
    
- 선택 필터는 “있을 때만 등식”으로 붙이는 동적 WHERE 유지(OR/`1=1` 금지) ✅
    
- `stat=2` 같은 필터가 있을 때 서브쿼리에  
    `USE INDEX (ix_voucher_regdate_seq)`를 유지하면 **정렬 인덱스에서 안 새고** 안정적 ✅
    
- 여전히 특정 케이스에서 느리면 보조 인덱스 후보: `(stat, regdate, seq_no)` (정말 자주 쓸 때만)
    

---

### 결론

지금 EXPLAIN 결과는 **좋습니다.**  
가시적으로 `Using filesort`를 지우고 싶다면 **옵션 B**처럼 바깥 정렬을 `t.regdate, t.seq_no`로 바꾸고 `t`에서 해당 컬럼을 함께 SELECT 하세요. 그렇지 않아도 성능상 문제는 없어요.