---
layout: default
title: "[MySQL] 세션변수"
parent: MySQL
nav_order: 1
---

# 세션 변수

MySQL에서 `@변수`와 `DECLARE 변수`는 비슷해 보이지만 성격이 전혀 달라요:

---

### 1. `@변수` (세션 변수, User-defined variable)

- **범위:** 현재 세션 전체 (프로시저 안/밖 어디서든 사용 가능).
    
- **선언 필요 없음:** 그냥 `SET @var := 값;` 또는 `SELECT ... INTO @var;` 하면 바로 생성됨.
    
- **타입 고정 없음:** 대입되는 값에 따라 동적으로 타입이 결정됨.
    
- **주 용도:** 쿼리 실행 중간에 값을 임시 저장하거나, 여러 쿼리에서 공유할 때 사용.
    

예시:
```sql
SET @cnt := 0;  
SELECT COUNT(*) INTO @cnt FROM orders WHERE status = 'READY';  
SELECT @cnt;  -- 세션 어디서든 접근 가능
```


---

### 2. `DECLARE 변수` (로컬 변수, Local variable)

- **범위:** **BEGIN ... END** 블록 내부 (Stored Procedure, Function, Trigger 안에서만 사용 가능).
    
- **선언 필수:** 반드시 `DECLARE 변수명 데이터타입;` 형식으로 선언해야 함.
    
- **타입 지정 필요:** `INT`, `VARCHAR(20)`, `DATE` 등 명시해야 함.
    
- **주 용도:** 프로시저 내부에서만 잠깐 쓰는 임시 변수.
    

예시:

```sql
DECLARE v_cnt INT DEFAULT 0;  
SELECT COUNT(*) INTO v_cnt FROM orders WHERE status = 'READY';  -- v_cnt는 프로시저가 끝나면 소멸됨
```


---

### ✅ 정리

- `@var`: 세션 단위, 선언 필요 없음, 타입 자유, 프로시저 밖에서도 유지됨.
    
- `DECLARE var`: 블록 단위, 선언 필수, 타입 고정, 블록 종료 시 사라짐.
    

---

👉 즉, 프로시저 안에서만 잠깐 쓰는 값이면 `DECLARE 변수`, 여러 쿼리에서 이어서 쓰거나 프로시저 밖에서도 활용해야 하면 `@변수`를 쓰면 돼요.