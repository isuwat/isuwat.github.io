---
layout: default
title: "[MySQL] NULLIF()"
parent: MySQL
nav_order: 3 
---

# NULLIF()

```sql
SET v_scope = NULLIF(TRIM(pi_scope), '');
```


이 구문은 **입력 파라미터 `pi_scope`를 정리해서 로컬 변수 `v_scope`에 넣는 처리**입니다.

---

## 🔎 동작 순서

1. **`TRIM(pi_scope)`**
    
    - 문자열 양쪽 공백 제거
        
    - 예: `' all '` → `'all'`
        
    - 예: `' '` → `''` (공백만 있으면 빈 문자열)
        
2. **`NULLIF(expr1, expr2)`**
    
    - `expr1 = expr2` 이면 `NULL` 반환
        
    - 아니면 `expr1` 반환
        
    
    → 즉, `TRIM(pi_scope)`가 `''` (빈 문자열)이면 `NULL`, 아니면 원래 값 유지
    
3. **대입**
    
    - 최종 결과를 `v_scope`에 넣음
        

---

## ✅ 의미

- `pi_scope`가 `NULL` 이거나 빈 문자열 `' '` 같은 경우 → `v_scope = NULL`
    
- `pi_scope`가 `'selected'` 또는 `'all'` → 그대로 유지
    

즉, **"빈 문자열은 NULL로 처리한다"** 는 로직이에요.

---

## 예시

|`pi_scope` 입력|`TRIM(pi_scope)`|결과 (`v_scope`)|
|---|---|---|
|`'selected'`|`'selected'`|`'selected'`|
|`' all '`|`'all'`|`'all'`|
|`''`|`''`|`NULL`|
|`' '`|`''`|`NULL`|
|`NULL`|`NULL`|`NULL`|

---

✅ **정리**  

```sql
SET v_scope = NULLIF(TRIM(pi_scope), '');  
```

👉 입력 값이 비어 있거나 공백만 있으면 `NULL`로 바꾸고, 정상 값이면 그대로 둔다.

---

원하시면 이 로직이 실제 프로시저 분기(`IF v_scope = 'all' ... ELSEIF v_scope = 'selected'`)에서 어떻게 동작하는지도 연결해서 설명드릴까요?

##### 나의 말:

pi_scope가 NULL이면 공백''과 같지 않은데 결과가 NULL인가?\

###### ChatGPT의 말:

좋은 질문이에요 👌 핵심은 **`NULLIF(expr1, expr2)`의 동작 규칙**입니다.

---

## 🔎 `NULLIF(expr1, expr2)` 규칙

- `expr1 = expr2` → `NULL` 반환
    
- `expr1 ≠ expr2` 또는 비교 불가 → `expr1` 반환
    

즉, 비교할 때 **`expr1`이 NULL이면, NULL을 그대로 반환**합니다.

---

## 🔎 지금 코드

```sql
SET v_scope = NULLIF(TRIM(pi_scope), '');
```


1. `pi_scope = NULL` 인 경우
    
    - `TRIM(NULL)` = `NULL`
        
    - `NULLIF(NULL, '')` → 비교할 때 `NULL = ''` 은 `UNKNOWN`
        
    - 따라서 결과는 `expr1` 즉 `NULL`
        

👉 최종적으로 `v_scope = NULL`

---

## 📌 예시 실행

```sql
SELECT TRIM(NULL);          -- NULL SELECT NULLIF(NULL, '');    -- NULL
```


---

✅ 결론  
`pi_scope`가 `NULL`이면 **그대로 `NULL` 유지**됩니다.  
즉, `pi_scope`가 NULL이어도 결과는 `NULL`이고, 빈 문자열 `''`과는 다르게 처리되지만 결과적으로 `v_scope`에는 NULL이 들어갑니다.