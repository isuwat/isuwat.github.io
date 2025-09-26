---
layout: default
title: "[MySQL] 쿼리지연-어드민 voucher 조회" 
parent: MySQL
nav_order: 23
---



# [Story] select_voucher 성능/스키마 정리 및 조회 컬럼 표준화

## 배경

- 대량 구간 조회에서 `ORDER BY ... LIMIT`가 느리고, OR 분기/포맷팅이 서버 부하 유발.
    
- 페이지네이션을 위해 “필터 일치 전체건수/합계”도 안정적으로 필요.
    

## 목표/범위

- (1) 전역 기간 조회 + 정렬/페이징 성능 상향
    
- (2) COUNT/SUM(동일 WHERE) 정확도/속도 확보
    
- (3) 조회 결과 컬럼의 “코드/라벨/표시” 분리로 API 스키마 명확화
    

---

## 변경사항(요약)

1. DB 인덱스 추가

```sql
CREATE INDEX ix_voucher_regdate_seq ON voucher (regdate, seq_no);

```

1. `select_voucher` 프로시저 리팩터
    
    - 동적 WHERE(“있을 때만 등식”) + 반열림 구간(`[start, end+1day)`)
        
    - 전체건수/합계는 **동일 WHERE**로 1회 실행
        
    - 목록은 **두 단계 페치**: “키만 정렬(LIMIT) → seq_no JOIN 본문”
        
2. Admin 조회 컬럼명 표준화(뷰/응답 스키마)
    
    |기존|변경|비고|
    |---|---|---|
    |`price`(문자열 포맷)|`price_fmt`|표시는 `price_fmt`, 정렬/계산은 숫자 `price` 사용|
    |`order_user_type`(라벨)|`order_user_type_label`|코드 `order_user_type`(숫자)와 분리|
    |`stat`(라벨)|`stat_label`|코드 `stat`(숫자)와 분리|
    
3. 수정 전/후 지연시간 측정 및 EXPLAIN 검증
    
    - p95 기준 전/후 비교(대표 필터/정렬/페이지 1·5·10)
        

---

## 세부 작업(하위 태스크)

- [DB] 인덱스 생성 + 통계 갱신
    
    -  `CREATE INDEX ix_voucher_regdate_seq ON voucher(regdate, seq_no);`
        
    -  `ANALYZE TABLE voucher;`
        
- [SP] `select_voucher` 동적 WHERE/두 단계 페치로 변경
    
    -  날짜 반열림 적용: `regdate >= :start AND regdate < :end_next`
        
    -  OR/`1=1` 제거, **필요 시에만** 등식 조건 추가
        
    -  전체건수/합계 쿼리 1회 실행(동일 WHERE)
        
    -  목록:

```sql
SELECT v.<필요컬럼들…>
FROM (
  SELECT s.seq_no
  FROM voucher s /* USE INDEX(ix_voucher_regdate_seq) 검토 */
  WHERE <동일 WHERE>
  ORDER BY s.regdate <ASC|DESC>, s.seq_no <ASC|DESC>
  LIMIT :offset, :size
) t
JOIN voucher v ON v.seq_no = t.seq_no
ORDER BY v.regdate <ASC|DESC>, v.seq_no <ASC|DESC>;
```

- [FE/Admin] `voucher_select.php` 응답 컬럼 명세 반영
    
    -  `price_fmt`, `order_user_type_label`, `stat_label` 사용
        
    -  정렬/계산은 숫자 `price`, 코드 `order_user_type/stat` 사용
        
- [QA] 성능/정확성 검증
    
    -  EXPLAIN: 서브쿼리 `type=range`, `key=ix_voucher_regdate_seq`, `Using filesort` 없음 확인
        
    -  대표 케이스 6종 p95: (전체/`user_id`/`restore_user_id`/`stat=2`/`serial_code`/`biz_id`) × (DESC/ASC) × (page=1/5/10)
        
    -  COUNT/SUM이 목록 WHERE와 일치하는지 랜덤 샘플 대조
        

---

## Acceptance Criteria

- [AC1] 대표 조회 시 p95 ≤ **~400ms**(상황에 맞게 팀 기준)
    
- [AC2] page 10에서도 p95 편차 **±20% 이내**(키셋/두 단계 페치 효과)
    
- [AC3] COUNT/SUM 결과가 동일 WHERE의 레코드 집합과 일치(무작위 표본 30건 대조 통과)
    
- [AC4] EXPLAIN 서브쿼리에서 정렬 인덱스 사용(`ix_voucher_regdate_seq`), filesort 미발생
    
- [AC5] Admin 화면 정상 렌더(숫자/라벨/포맷 분리), 정렬·필터 정확
    

---

## 성능 측정 계획(샘플)

- 기간: `@start='2025-07-01'`, `@endNxt='2025-10-02'`
    
- SQL 샘플(전/후 동일):

```sql
-- 목록 키 서브쿼리
EXPLAIN
SELECT s.seq_no
FROM voucher s USE INDEX (ix_voucher_regdate_seq)
WHERE s.regdate >= @start AND s.regdate < @endNxt
  /* 있을 때만 등식: user_id=?, stat=?, ... */
ORDER BY s.regdate DESC, s.seq_no DESC
LIMIT 0, 50;

-- 전체건수/합계
EXPLAIN
SELECT COUNT(*), SUM(price)
FROM voucher
WHERE regdate >= @start AND regdate < @endNxt
  /* 동일 WHERE */
;
```

## 릴리즈/롤백

- 순서: 인덱스 생성 → 프로시저 배포 → FE 반영 → 통계 최신화 → 모니터링
    
- 롤백:
    
    - SP: 이전 버전 파일로 즉시 롤백
        
    - FE: 컬럼명 구버전 호환 분기(임시) 유지 가능
        
    - 인덱스: 문제 시 `DROP INDEX ix_voucher_regdate_seq ON voucher;` (권고: 원인 파악 전 드롭은 지양)
        

---

## 리스크 & 완화

- 옵티마이저가 `stat` 등 다른 인덱스로 새는 이슈 → 서브쿼리에서 `USE INDEX(ix_voucher_regdate_seq)` 옵션 유지/토글
    
- 인덱스 증가로 쓰기 비용↑ → 모니터링(버퍼풀 적중률/쓰기 딜레이), 불필요 인덱스 정리
    
- FE 컬럼명 변경에 따른 호환 → 배포 전후 1 스프린트 동안 구/신 병행 파싱 분기 유지
    

---

## 산출물

- SQL: 인덱스 DDL, 리팩터된 `select_voucher`(동적 WHERE + 두 단계 페치 + COUNT/SUM)
    
- 문서: EXPLAIN 캡처, 전/후 p95 비교표, Admin 응답 스키마 변경 노트