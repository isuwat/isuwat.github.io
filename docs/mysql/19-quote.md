---
layout: default
title: "[MySQL] QUOTE()" 
parent: MySQL
nav_order: 19
---



## `QUOTE()`는 뭐냐면…

MySQL의 **문자열 리터럴 이스케이프 함수**예요. 동적 SQL을 만들 때 안전하고 정확한 문자열 리터럴을 넣으려고 씁니다.

- 동작: 인자를 **작은따옴표로 감싸고** 내부의 `'`, `\`, NUL, LF, CR, Ctrl-Z 등을 이스케이프해서 반환.
    
- 예시:

```sql
SELECT QUOTE("O'Reilly");      -- 결과: 'O\'Reilly'
SELECT QUOTE('2025-09-26');    -- 결과: '2025-09-26'
SELECT QUOTE(NULL);            -- 결과: NULL

```

우리가 동적 WHERE를 만들 때

```sql
SET v_where = CONCAT('regdate >= ', QUOTE(v_start), ' AND regdate < ', QUOTE(v_end));
SET v_where = CONCAT(v_where, ' AND user_id = ', QUOTE(pi_keyword));

```

처럼 써서 **문법 오류/SQL 인젝션 위험**을 줄이고 있습니다.  
(가능하면 `PREPARE ...; EXECUTE ... USING @p1,@p2` 바인딩도 좋지만, 조건 자체를 조립해야 할 때 `QUOTE()`가 편리합니다.)