---
layout: default
title: "[MySQL] sp 지연 분석"
parent: MySQL
nav_order: 7 
---


# 단 건 INSERT 느려짐 분석



# 어디서 시간이 새는가?

## 1) 트랜잭션 중 `SIGNAL`만 있고 `ROLLBACK`이 없음 → 락 홀드/경합 유발

`START TRANSACTION;` 이후 여러 검증에서 `SIGNAL`로 예외만 던지고 **실제 ROLLBACK을 호출하지 않습니다.**  
예외가 올라가도 세션이 롤백을 명시하지 않으면 **락이 오래 유지**되어 이후 요청들이 `member` 또는 `voucher`에서 대기할 수 있어요. 이건 단건 느려짐의 흔한 원인입니다.

### 빠른 해결

- 모든 오류 경로에서 **명시적 ROLLBACK** 수행
    
- 혹은 **핸들러**로 일괄 롤백
    

```sql
-- 프로시저 상단에 공통 핸들러 추가
DECLARE EXIT HANDLER FOR SQLEXCEPTION
BEGIN
    ROLLBACK;
    RESIGNAL; -- 원래 에러 다시 던짐
END;

```

또는 각 오류 지점에서:

```sql
SET po_result_code = -3;
SET po_err_msg = '...';
ROLLBACK;
SIGNAL SQLSTATE '45000';

```

## 2) 읽고(SELECT) 나중에 갱신(UPDATE) → 경쟁 시 대기/데드락

현재 흐름:

- `SELECT user_voucher,user_type FROM member WHERE user_id = ?;` (락 없음)
    
- 검증 후 `UPDATE member SET user_voucher = ... WHERE user_id = ?;`
    

동시에 두 요청이 같은 `user_id`를 다루면 **서로 갱신 경쟁**이 납니다.  
행을 먼저 고정해 버리면(락) 대기가 짧고 예측 가능해져요.

### 빠른 해결

- 검증 직전에 **행 잠금**: `SELECT ... FOR UPDATE`
    

```sql
SELECT user_voucher, user_type
INTO v_member_cur_amt, v_user_type
FROM member
WHERE user_id = pi_user_id
FOR UPDATE;  -- 여기!

```


- `member.user_id`에 **PK/UNIQUE 인덱스**가 꼭 있어야 `FOR UPDATE`가 즉시 해당 행만 잡습니다.
    

## 3) `serial_code` UNIQUE 인덱스가 “길고 비싼” 비교를 함

- 현재 `serial_code`가 `varchar(100) utf8mb4_general_ci` + **UNIQUE**
    
- 실제 값은 MD5(32hex) + 하이픈 3개 ≈ **35자 ASCII**인데, utf8mb4/CI 비교는 비용이 큽니다.
    
- **인덱스 키가 길수록** 삽입 시 B-Tree 갱신 비용이 커지고, UNIQUE 중복 검사도 느립니다.
    

### 빠른 해결 (권장 순서)

1. **고정 길이 & ASCII로 축소**
    

```sql
ALTER TABLE voucher 
  MODIFY serial_code CHAR(36) 
    CHARACTER SET ascii COLLATE ascii_general_ci NOT NULL,
  DROP INDEX serial_code_UNIQUE,
  ADD UNIQUE KEY serial_code_UNIQUE (serial_code);

```


2. 더 빠르게: **바이너리 16바이트**로 저장/인덱스
    
    - 생성 시: `UNHEX(REPLACE(UUID(),'-',''))` 혹은 `UNHEX(MD5(...))`
        

```sql
ALTER TABLE voucher 
  MODIFY serial_code BINARY(16) NOT NULL,
  DROP INDEX serial_code_UNIQUE,
  ADD UNIQUE KEY serial_code_UNIQUE (serial_code);

```


- 프로시저:
    

```sql
-- MD5 기반(현 로직 유지 시)
SET v_serial_code_bin = UNHEX(MD5(CONCAT(pi_user_id, v_now6, pi_voucher_open_money, RAND())));
INSERT INTO voucher (serial_code, price, stat, expire_date, regdate, user_id, user_name, order_user_type, balance)
VALUES (v_serial_code_bin, ...);

```


- 조회/표시는 `HEX(serial_code)`로 가독성 확보.
    

> **효과**: 인덱스 키가 36바이트(ASCII) → **16바이트(바이너리)**로 줄어들고, 대조도 **바이너리**라 매우 빠릅니다. (utf8mb4 CI 대비 큰 차이)

## 4) `trdno` UNIQUE도 같은 최적화 가능

`trdno`가 외부 거래번호라면 대부분 ASCII입니다. 길이가 과한 편(100)이고 utf8mb4/CI예요.  
가능하면 **ASCII / 필요 길이로 축소**(예: 40~64) + UNIQUE 유지 권장.

```sql
ALTER TABLE voucher 
  MODIFY trdno VARCHAR(64) CHARACTER SET ascii COLLATE ascii_general_ci DEFAULT NULL,
  DROP INDEX trdno_UNIQUE,
  ADD UNIQUE KEY trdno_UNIQUE (trdno);

```


## 5) 기타 마이너/유용한 팁

- `NOW()`/`NOW(6)`를 **한 번 변수에 담아 재사용**(동일 트랜잭션 타임스탬프 보장 + 함수 호출 감소)
```sql
SET v_now = NOW(6);
...
INSERT ... regdate = v_now;

```
    
     
- `expire_date` 계산도 동일하게 변수화.
    
- `AUTO_INCREMENT`는 InnoDB 8.0 기본이 interleaved(= 경합 적음)라 그대로 OK.
    
- `voucher`는 PK가 증가형이라 **클러스터드 인덱스 페이지 분할 부담이 적음** → 좋습니다.
    
- `FOREIGN KEY`가 없으니 FK 검증 비용은 없음.
    

---

# 수정 예시(핵심만 반영한 스니펫)

```sql
-- 1) 공통 핸들러
DECLARE EXIT HANDLER FOR SQLEXCEPTION
BEGIN
    ROLLBACK;
    RESIGNAL;
END;

-- 2) 검증은 트랜잭션 밖에서 대부분 수행(가능한 경우)
--    금액/한도 등 “상태 의존” 검증만 트랜잭션 안으로

START TRANSACTION;

-- 3) 필요한 행 먼저 잠금
SELECT user_voucher, user_type
INTO v_member_cur_amt, v_user_type
FROM member
WHERE user_id = pi_user_id
FOR UPDATE;

-- 4) 재검증(잔액 등)
IF v_member_cur_amt IS NULL THEN
    SET po_result_code = -2; SET po_err_msg = '회원이 존재하지 않습니다.';
    ROLLBACK; SIGNAL SQLSTATE '45000';
END IF;

-- (중략: 한도/금액 검증)

-- 5) 갱신
SET v_member_result_amt = v_member_cur_amt - pi_voucher_open_money;
UPDATE member
SET user_voucher = v_member_result_amt
WHERE user_id = pi_user_id;  -- (PK/UNIQUE 인덱스 전제)

IF ROW_COUNT() = 0 THEN
    SET po_result_code = -8; SET po_err_msg = 'PIN 차감 오류';
    ROLLBACK; SIGNAL SQLSTATE '45000';
END IF;

-- 6) 코드 생성(이 예시는 16바이트 바이너리)
SET v_now6 = NOW(6);
SET v_serial_code_bin = UNHEX(MD5(CONCAT(pi_user_id, v_now6, pi_voucher_open_money, RAND())));

INSERT INTO voucher(serial_code, price, stat, expire_date, regdate, user_id, user_name, order_user_type, balance)
VALUES (v_serial_code_bin, pi_voucher_open_money, v_stat, v_expire_date, v_now6, pi_user_id, pi_user_name, v_order_user_type, v_member_result_amt);

COMMIT;

-- OUT 파라미터는 표시용으로 HEX 포맷을 돌려줄 수도 있음
SET po_serial_code = UPPER(HEX(v_serial_code_bin));

```


---

# 체크리스트 (우선순위)

1. **핸들러로 ROLLBACK 보장**(가장 중요)
    
2. `member` 조회를 **`FOR UPDATE`로 잠금**
    
3. `serial_code`를 **BINARY(16) UNIQUE** 또는 **CHAR(36) ASCII UNIQUE**로 축소
    
4. `trdno`도 **ASCII + 길이 축소**
    
5. `NOW(6)` 등 **한 번 계산해 재사용**
    

이 5가지만 반영해도 단건 INSERT 지연이 유의미하게 줄 가능성이 높습니다.

