---
layout: default
title: "[MySQL] sp 지연 진단"
parent: MySQL
nav_order: 8
---


# 프로시저 지연 진단

# 1) 원인 빠르게 압축 체크

- **보조 인덱스가 많음**: INSERT 때 모든 인덱스를 갱신 → 느림.
    
- **랜덤 PK(예: UUID v4)**: 클러스터드 인덱스 페이지 분할·랜덤 I/O 증가.
    
- **UNIQUE/PK 중복 체크 비용**: 긴/여러 컬럼일수록 비쌈.
    
- **FOREIGN KEY**: 부모 테이블 조회(인덱스 없으면 치명적).
    
- **트리거/스토어드 함수**: 숨은 작업 시간 잡아먹음.
    
- **Autocommit=ON**: 매 건 fsync → 디스크 대기.
    
- **바이너리 로그/복제 설정**: `sync_binlog`, semisync 대기.
    
- **버퍼 풀/IO 한계**: `innodb_buffer_pool_size` 부족, 디스크 IOPS 낮음.
    
- **락 대기**: 같은 키/갭에 대한 잠금 경합.
    
- **네트워크 왕복**: DB는 빠른데 애플리케이션↔DB RTT가 큼(특히 TLS).
    

# 2) 어디서 막히는지 진단

- **가장 먼저**: 서버에서 대기 유형 보기

```sql
-- 최근 느린 문장 유형
SELECT * FROM sys.statements_with_runtimes_in_95th_percentile
WHERE digest_text LIKE 'INSERT%';

-- 대기/락 확인 (필요 시 Performance Schema 활성화)
SELECT EVENT_NAME, SUM_TIMER_WAIT
FROM performance_schema.events_waits_summary_global_by_event_name
ORDER BY SUM_TIMER_WAIT DESC
LIMIT 20;

-- 락 대기
SELECT * FROM sys.innodb_lock_waits ORDER BY wait_time DESC LIMIT 10;

```
    
       
- **InnoDB 상태/IO**

```sql
SHOW ENGINE INNODB STATUS\G
-- history list length가 길거나, buffer pool miss/IO pending 많으면 자원 부족 신호

```
    
       
- **트리거/제약 존재 여부**

```sql
SELECT TRIGGER_NAME, EVENT_MANIPULATION FROM information_schema.TRIGGERS
WHERE EVENT_OBJECT_TABLE='your_table';
SHOW CREATE TABLE your_table\G

```
    
       
- **인덱스 과다 확인**

```sql
SHOW INDEX FROM your_table;

```
    
       
- **설정(지연 민감 항목)**

```sql
SHOW VARIABLES WHERE VARIABLE_NAME IN
('innodb_flush_log_at_trx_commit','sync_binlog','binlog_format','innodb_buffer_pool_size');

```
   
   

# 3) 바로 적용 가능한 개선책 (단건 INSERT에도 효과)

## 테이블/스키마 측면

- **PK를 단조 증가형으로**: UUID v4 → **UUID v7/ULID/Auto Increment** 등으로 변경하면 페이지 분할/랜덤 I/O 급감.
    
- **PK는 “짧고 단순하게”**: 긴 문자 PK·복합 PK 지양.
    
- **불필요한 보조 인덱스 줄이기**: 사용 빈도 낮은 인덱스 삭제.
    
- **FOREIGN KEY의 부모 키에 인덱스 필수**: 없으면 즉시 추가.
    
- **트리거 최소화/로직 앱단 이동**: 특히 외부 조회/함수 호출 제거.
    

## 트랜잭션/로그 설정(안전성↔성능 트레이드오프)

- **Autocommit OFF + 배치**: 정말 단건만 온다면 앱에서 **N건 모아 한 트랜잭션**으로.
    
- **`innodb_flush_log_at_trx_commit=2`**: 내구성 약간 낮추고 지연 크게 절감(테스트/비핵심 서비스에 우선).
    
- **`sync_binlog=1` 또는 0**: 1은 안전, 0은 성능. 요구사항에 맞춰 조정.
    
- **복제 세미동기라면**: 단건 지연이 크면 임시로 비활성/대기 타임아웃 조정.
    

## 리소스/인프라

- **버퍼 풀 확대**: 메모리 여유 시 `innodb_buffer_pool_size`를 RAM의 60~75% 권장.
    
- **디스크 IOPS/지연** 개선: 로컬 NVMe, 전용 볼륨, 파일시스템 마운트 옵션 점검.
    
- **DB와 앱의 네트워크 RTT 줄이기**: 같은 VPC/리전, 커넥션 풀/준비된문 사용.
    

# 4) 체크리스트 (단건 INSERT 튜닝 우선순위)

1. `SHOW INDEX`로 인덱스 수 과다? → **정리**
    
2. PK가 랜덤? → **단조 증가 키로 전환(예: UUID v7)**
    
3. 트리거/함수? → **비활성/이전**
    
4. FK 부모키 인덱스 유무? → **추가**
    
5. Autocommit ON? → **앱단에서 배치 커밋**
    
6. 설정 값 점검
    
    - 지연 크면: `innodb_flush_log_at_trx_commit=2`(테스트 후), `sync_binlog` 조정
        
    - `binlog_format`은 일반적으로 **ROW** 권장(정합성), 지연 크면 전체 구조 재검토
        
7. 버퍼 풀/IO/네트워크 병목? → **자원 증설 또는 배치 경로 최적화**
    

# 5) 샘플 성능 비교(애플리케이션 측)

- 단건 반복:

```sql
-- Autocommit ON에서
INSERT INTO t(...) VALUES (...);  -- 매번 fsync

```
    
       
- 배치/준비된 문:

```sql
START TRANSACTION;
INSERT INTO t(col1,col2,...) VALUES 
  (...), (...), (...);  -- 여러 건을 한 번에
COMMIT;

```

  
- 랜덤 PK → 시간순 PK:
    
    - **UUID v4** 대신 **UUID v7**(시간순) 또는 AUTO_INCREMENT 사용.