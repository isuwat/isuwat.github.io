---
layout: default
title: "[MySQL] USE INDEX" 
parent: MySQL
nav_order: 20
---



### USE INDEX (ix_voucher_regdate_seq) <- 넣는 이유?

짧게 답하면: **꼭 넣을 필요는 없지만, “키만 뽑는 서브쿼리”에선 넣는 걸 추천**합니다.

왜냐면:

- 인덱스가 잘 잡히면(= `EXPLAIN`에 `key=ix_voucher_regdate_seq`, `Using filesort` 없음) **생략해도 OK**.
    
- 하지만 우리 케이스처럼 필터(`stat=2` 등)가 붙을 때 옵티마이저가 엉뚱한 인덱스를 골라 `ALL + filesort`로 가는 걸 **여러 번 봤죠**. 이럴 땐 `USE INDEX (ix_voucher_regdate_seq)`가 **플랜을 안정화**시켜줍니다.  
    (먼저 `USE INDEX`로 ‘부드럽게’ 유도 → 그래도 삐끗하면 `FORCE INDEX` or 보조 인덱스 추가)
    

실전 가이드:

1. 기본은 **생략**하고 `EXPLAIN` 확인
    
    - 서브쿼리(id=derived)가 `type=range`, `key=ix_voucher_regdate_seq`, `filesort 없음` → 계속 생략 가능
        
2. 하나라도 어긋나면 **`USE INDEX` 추가**
    
    - 여전히 새면 → `FORCE INDEX` 또는 `(stat, regdate, seq_no)` 같은 보조 인덱스 신설
        
3. 동적 SQL이라면 **힌트를 토글**할 수 있게:

```sql
SET @from_tbl := 'FROM voucher s';
IF pi_hint_regdate = 1 THEN
  SET @from_tbl := 'FROM voucher s USE INDEX (ix_voucher_regdate_seq)';
END IF;

SET @list_sql = CONCAT(
  'SELECT s.seq_no ', @from_tbl, ' WHERE ', v_where, v_order_by,
  ' LIMIT ', pi_offset, ', ', pi_page_size
);

```

요약: **우선 생략하고 계획이 안정적이면 그대로**, 아니라면 **서브쿼리에서만 `USE INDEX`로 고정**하세요. 오래 가는 안전한 절충안입니다.