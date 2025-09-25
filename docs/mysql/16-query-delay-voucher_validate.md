---
layout: default
title: "[MySQL] 쿼리지연-voucher 검증" 
parent: MySQL
nav_order: 16
---



로그만 보면 행 스캔은 거의 없는데(Examined: 3), 벽시계 시간이 1.72s라 “쿼리 자체”보다 **트랜잭션/스토리지/환경 요인**일 확률이 큽니다. `validate_voucher_pin`의 흐름을 보면 초반에 무조건 `START TRANSACTION`을 하고, 각 검증 실패 케이스에서는 `LEAVE`로 빠져나오며 커밋이 없습니다(성공시에만 `COMMIT`). 이 패턴이 특정 상황에서 지연을 유발할 수 있어요.

아래 순서로 원인 점검→빠른 개선을 해보세요.

# 원인 후보 (우선순위)

1. **불필요한 트랜잭션 시작 시점**
    

- 검증 실패(예: biz_partner.id 없음, 핀 없음, stat≠1)에서도 이미 트랜잭션이 열린 상태로 `LEAVE` 합니다. 쓰기 작업이 없더라도, 트랜잭션 생성·정리 비용, Group Commit 경합, binlog/redo 경합 등으로 **간헐 지연**이 생길 수 있습니다. 이 SP는 성공 경로에서만 `COMMIT`을 호출하고, 실패 경로는 `LEAVE`로만 빠집니다.
    

2. **콜드 스타트/버퍼 미적중**
    

- 서버 재시작 직후, 또는 일정 시간 미사용 후 첫 호출이라면 `biz_partner`, `voucher` 페이지를 메모리로 올리는 IO 비용 때문에 첫 1~2초 정도 느릴 수 있습니다. 다음 호출들은 즉시(수 ms) 나오는지 확인하세요.
    

3. **Group Commit / fsync 지연**
    

- 성공 경로에서 `UPDATE voucher …` 후 `COMMIT`을 하는 경우, 동시 트랜잭션과의 그룹 커밋 대기 때문에 ms~수백 ms 이상 튈 수 있습니다. (이번 케이스가 성공 경로였는지는 OUT 값으로 확인 필요.)
    

4. **메타데이터 락·다른 세션 경합**
    

- 동시 DDL/대량 DML 등으로 잠깐 멈칫할 수 있습니다. 이번 로그에서는 `Lock_time: 0`이라 가능성은 낮지만, Performance Schema로 세부 이벤트를 확인해보는 게 확실합니다.
    

# 바로 적용 가능한 개선 (코드 변경 최소)

1. **트랜잭션 시작을 “검증 통과 후”로 이동** (권장-효과 큼)
    

- 현재: 시작하자마자 `START TRANSACTION;` → 여러 검증 → 실패 시 `LEAVE` (커밋/롤백 없음)
    
- 개선:
    
    - `biz_partner.id` 검증 통과
        
    - `voucher.serial_code` 존재 + `stat = 1` 검증 통과
        
    - **여기서 `START TRANSACTION`**
        
    - `UPDATE voucher …` → 실패 시 `SIGNAL` → 핸들러에서 ROLLBACK
        
    - 성공 시 `COMMIT`  
        → 실패 케이스에서 **트랜잭션 자체가 열리지 않으므로** 군더더기 대기 요소가 사라집니다. (현재 핸들러는 SQLEXCEPTION에서만 ROLLBACK을 호출하고 일반 실패는 `LEAVE`만 합니다. 이 구조와도 더 잘 맞습니다.)
        

2. **실패 경로 정리**
    

- 혹시라도 앞으로 실패 경로에서 트랜잭션이 열린 상태가 될 수 있다면, `IF @@tx_isolation…` 같은 체크 대신 **트랜잭션을 늦게 시작**하는 것이 안전합니다(위 1로 해결).
    

3. **정확한 원인 확인용 타이밍 로깅(경량)**
    

- 디버깅 일시용으로 단계별 시간을 OUT 변수로 내보내거나, 임시 로그 테이블에 `NOW(6)` 기준 차이를 남겨 병목을 특정하세요. 예:
    
    - T1: SP 진입 직후
        
    - T2: biz_partner 조회 후
        
    - T3: voucher 조회 후
        
    - T4: (성공 경로에서만) UPDATE 후
        
    - T5: COMMIT 후  
        → 어느 구간이 오래 걸리는지 즉시 보입니다.  
        (현 SP 흐름: biz_partner 조회 → voucher 조회 → stat 분기 → 성공 시 UPDATE+COMMIT)
        

4. **운영 환경 체크**
    

- 콜드스타트 의심 시 같은 입력으로 연속 3회 호출하여 2~3번째가 빨라지는지 비교.
    
- 동시간대 InnoDB 상태(redo flush, checkpoint, log write latency), binlog 모드, 디스크 IO 스파이크 확인.
    
- 가능하다면 `skip_name_resolve=ON`, 불필요한 DNS 역조회 회피(주로 접속 단계지만 환경 전체 지연이 섞일 수 있어요).
    

# 인덱스/쿼리 적합성 재확인

- 본 SP는 **정확히** `biz_partner.id = ?` 단건 조회, `voucher.serial_code = ?` 단건 조회, 그리고 업데이트(성공 시)에만 닫힙니다.
    
- 제공하신 인덱스 세트에서 `serial_code`는 **UNIQUE**라 단건 탐색/갱신 최적입니다. 이번 1.72s는 인덱스 부족이 아닌 **환경/흐름** 이슈로 보는 게 타당합니다.
    

# 패치 예시(요지)

- 아래처럼 “검증 통과 후에만” 트랜잭션을 시작하도록 위치 변경:
```sql
-- (검증 전) START TRANSACTION;  -- 삭제

START TRANSACTION;  -- ← 여기로 이동

UPDATE voucher
	...
 WHERE serial_code = pi_serial_code;

-- 에러 핸들러는 SQLEXCEPTION에서 ROLLBACK 유지
COMMIT;

```

이 한 줄 위치 변경만으로도, **실패 케이스의 호출 지연**은 눈에 띄게 줄어듭니다. 성공 케이스의 지연은 주로 커밋 경합/스토리지에 좌우되므로, 그때는 InnoDB/바이너리로그 튜닝 포인트를 같이 보시면 됩니다.