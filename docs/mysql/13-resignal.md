---
layout: default
title: "[MySQL] RESIGNAL"
parent: MySQL
nav_order: 13
---


# RESIGNAL



`RESIGNAL`은 **에러 핸들러 안에서 “현재 예외를 다시 던지는(rethrow)”** 명령이에요.  
말 그대로 _원래 발생한 오류(또는 SIGNAL로 만든 오류)를 롤백 같은 정리 작업 후, 호출자에게 그대로 전달_하겠다는 뜻입니다.

---

## 왜 쓰나요?

- 핸들러에서 `ROLLBACK` 같은 **정리 작업은 수행**하되,  
    에러 자체는 **삼키지 말고 바깥으로 전달**하려고 씁니다.
    
- `RESIGNAL`이 없으면, 핸들러가 오류를 “처리한 것”으로 간주되어 **성공처럼 끝나버릴 수** 있어요(특히 `EXIT` 핸들러). 그럼 클라이언트는 실패를 못 느낍니다.
    

---

## 전형적인 패턴

```sql
DECLARE EXIT HANDLER FOR SQLEXCEPTION
BEGIN
    ROLLBACK;   -- 정리
    RESIGNAL;   -- 원래 오류(혹은 우리가 만든 오류)를 그대로 다시 던짐
END;

```


동작 순서:

1. 어떤 SQL에서 에러 발생 →
    
2. 핸들러로 점프 → `ROLLBACK` 수행 →
    
3. `RESIGNAL`로 **같은 SQLSTATE/메시지**를 상위(호출자)에게 다시 전달 →
    
4. 호출자는 실패를 정확히 감지
    

---

## `SIGNAL` vs `RESIGNAL`

- `SIGNAL` : **새 예외를 발생**시킴 (예: `SIGNAL SQLSTATE '45000' SET MESSAGE_TEXT='...'`)
    
- `RESIGNAL` : **현재 예외를 재발생**시킴(핸들러 내부에서만 사용).  
    필요하면 일부 속성만 바꿔서 던질 수도 있어요.
    
```sql
-- 원본 예외 그대로
RESIGNAL;

-- 메시지만 바꿔서 재던지기
RESIGNAL SET MESSAGE_TEXT = '처리 중 오류가 발생했습니다';

-- SQLSTATE/번호/메시지 모두 지정(원본 덮어쓰기)
RESIGNAL SQLSTATE '45000' SET MYSQL_ERRNO = 10001, MESSAGE_TEXT = '사용자 정의 오류';

```


> 참고: `RESIGNAL`은 **핸들러 내부에서만** 쓸 수 있습니다. 밖에서 쓰면 오류가 납니다.

---

## 만약 `RESIGNAL`을 빼면?

아래처럼 핸들러가 **오류를 “먹고”** 정상 흐름처럼 종료될 수 있습니다.

```sql
DECLARE EXIT HANDLER FOR SQLEXCEPTION
BEGIN
    ROLLBACK;
    -- RESIGNAL이 없음 → 오류를 처리한 것으로 간주
END;

```


그 결과, 클라이언트는 **OK로 보거나**(상황에 따라) 최소한 **원래 오류 정보**를 못 받게 됩니다.

---

## 당신 프로시저에 적용 예

프로시저 맨 위쪽(주 블록)에 넣어두면, 중간에 `SIGNAL` 하든 실제 SQL 오류가 나든 **항상 롤백 + 재던지기**가 보장됩니다.

```sql
DECLARE EXIT HANDLER FOR SQLEXCEPTION
BEGIN
    ROLLBACK;
    RESIGNAL; -- 원래 에러 다시 던짐
END;

```


요약:  
**“RESIGNAL = (핸들러에서) 에러를 정리 후, 원래 에러(또는 약간 수정한 에러)를 다시 밖으로 던져라.”**


#### 핸들러 자체가 EXIT인데 다시 재던지기 하는 이유는?

## 왜 `EXIT` 핸들러인데 `RESIGNAL`을 또 쓰나?

- **`EXIT`**은 _제어 흐름_을 말합니다. “에러가 나면 이 핸들러를 실행하고, 그 **블록을 빠져나가라**”는 뜻이에요.
    
- **하지만 에러 자체는 ‘처리된 것’으로 간주**됩니다. 즉, 핸들러에서 아무 것도 던지지 않으면 **원래 에러가 호출자까지 전달되지 않습니다.**
    
- 그래서 **`RESIGNAL`로 에러를 다시 던져**야, 호출자(애플리케이션/상위 프로시저)가 “실패”로 인식합니다.
    

요약: `EXIT` = 흐름 종료, `RESIGNAL` = 에러 전파. 역할이 달라서 둘 다 필요해요.

---

## 예시 1) `RESIGNAL`이 없어서 에러가 **삼켜지는** 경우

```sql
DELIMITER //
CREATE PROCEDURE p_no_resignal()
BEGIN
  DECLARE EXIT HANDLER FOR SQLEXCEPTION
  BEGIN
    ROLLBACK;          -- 정리는 함
    -- RESIGNAL 없음 → 원래 에러는 여기서 ‘처리됨’
  END;

  START TRANSACTION;
  INSERT INTO t(pk) VALUES (1);
  INSERT INTO t(pk) VALUES (1);  -- PK 중복: 에러 발생 → 핸들러 진입 → ROLLBACK → 블록 종료
  COMMIT;                        -- 위 INSERT 실패했지만, 에러가 던져지지 않아 프로시저는 정상 종료
END//
DELIMITER ;

```

**호출 결과(클라이언트 입장):**
```csharp
mysql> CALL p_no_resignal();
Query OK, 0 rows affected

```

- 실제로는 롤백되어 아무 것도 안 들어갔는데, 호출자에겐 **성공처럼 보일 수 있음** → 로직 혼선, 잘못된 성공 처리 위험.
    

---

## 예시 2) `RESIGNAL`로 **롤백 후 에러를 재전달**하는 경우

```sql
DELIMITER //
CREATE PROCEDURE p_with_resignal()
BEGIN
  DECLARE EXIT HANDLER FOR SQLEXCEPTION
  BEGIN
    ROLLBACK;         -- 정리
    RESIGNAL;         -- 원래 에러(예: 1062 Duplicate) 그대로 호출자에게 전달
  END;

  START TRANSACTION;
  INSERT INTO t(pk) VALUES (1);
  INSERT INTO t(pk) VALUES (1);  -- 여기서 에러 → 핸들러 → 롤백 → RESIGNAL → 호출자는 실패로 인식
  COMMIT;
END//
DELIMITER ;

```


**호출 결과:**
```scss
mysql> CALL p_with_resignal();
ERROR 1062 (23000): Duplicate entry '1' for key 'PRIMARY'

```


- 호출자가 확실히 **실패를 감지**하니, 재시도/에러 메시지/경고 로그 등 적절한 대응이 가능합니다.
    

---

## 살짝 더: 메시지를 바꿔서 던지고 싶다면

원본 에러를 그대로 쓰지 않고 **사용자 정의 메시지**로 바꿔서 재던질 수도 있어요.

```sql
DECLARE EXIT HANDLER FOR 1062  -- 중복키 오류만 잡음 
BEGIN   ROLLBACK;   
RESIGNAL SQLSTATE '45000'     
SET MESSAGE_TEXT = '중복된 핀번호입니다. 다시 시도하세요'; 
END;
```

---

## 결론

- `EXIT 핸들러`는 “여기까지만 실행”이라는 **흐름 제어**일 뿐, 에러를 자동으로 전파하지 않습니다.
    
- **`ROLLBACK; RESIGNAL;`** 패턴을 쓰면 정리는 하고, **실패도 정확히 호출자에게 알릴 수 있어** 안전합니다.
    
- 지금 작성하신 프로시저에도 이 패턴을 넣으면 **락 정리 + 실패 전파**가 보장되어 단건 INSERT 지연/오판을 줄이는 데 도움이 됩니다.
- 
## 그래서 `RESIGNAL`이 필요한 이유

- 핸들러에서 `ROLLBACK`만 하고 **`RESIGNAL`을 안 하면 에러가 “처리된 것”**으로 간주되어, 바깥 흐름이 계속될 수 있습니다(케이스 B처럼 COMMIT 실행).
    
- `RESIGNAL`을 쓰면 **롤백 후 에러를 호출자까지 전파**하므로, 바깥 블록에서도 더 이상 진행하지 않고(호출자 입장에선 에러) 안전합니다.
    

권장 패턴:
```sql
DECLARE EXIT HANDLER FOR SQLEXCEPTION
BEGIN
  ROLLBACK;
  RESIGNAL;  -- 정리 후 에러 재전파 (바깥으로 실패를 알려 흐름 중단)
END;

```

## TL;DR

- **블록** = 핸들러가 선언된 `BEGIN … END` 범위.
    
- **EXIT 핸들러** = 그 블록 안의 남은 문장(예: COMMIT)을 **건너뛰고 블록 밖으로** 나감.
    
- COMMIT이 **같은 블록**이면 스킵, **바깥 블록**이면 실행될 수 있음.
    
- 원치 않는 진행을 막으려면 **`ROLLBACK; RESIGNAL;`** 조합을 쓰세요.