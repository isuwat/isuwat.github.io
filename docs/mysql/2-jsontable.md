---
layout: default
title: "[MySQL] JSON_TABLE"
parent: MySQL
nav_order: 2
---

# JSON_TABLE

좋은 질문이에요 👍 `JSON_TABLE` 구문을 풀어서 설명드리면:
`JSON_TABLE()`은 **MySQL 8.0.4 이상**에서만 지원됩니다.

```sql
FROM JSON_TABLE(
    p_cart_ids_json,         -- JSON 배열 파라미터 (예: '[101,102,105]')
    '$[*]'                   -- JSON 경로: 배열의 모든 요소
    COLUMNS (
        cid BIGINT PATH '$'  -- 각 요소를 BIGINT로 읽어서 cid 컬럼으로 반환
    )
) AS j

```

---

### 1. `p_cart_ids_json`

- 스토어드 프로시저에 `IN p_cart_ids_json JSON` 파라미터로 전달받은 값입니다.
    
- 예시: `'[101,102,105]'`
    

### 2. `'$[*]'`

- JSON Path 문법입니다.
    
- `$` = JSON 루트
    
- `[*]` = 배열의 모든 원소
    
- 즉, JSON 배열 안의 모든 값을 순회하겠다는 뜻입니다.
    

### 3. `COLUMNS (cid BIGINT PATH '$')`

- 배열 원소 하나하나를 **컬럼으로 매핑**하는 규칙을 정의합니다.
    
- `cid BIGINT` → 결과 테이블에 `cid`라는 BIGINT 컬럼을 만듭니다.
    
- `PATH '$'` → 각 원소 그 자체(`$` = 현재 요소)를 매핑합니다.
    

즉, `[101,102,105]`가 들어오면 아래와 같은 가상 테이블이 생성됩니다:

|cid|
|---|
|101|
|102|
|105|

### 4. `AS j`

- 위에서 만든 가상 테이블에 `j`라는 별칭(alias)을 붙입니다.
    
- 이후 `SELECT j.cid FROM ...` 식으로 쓸 수 있습니다.
    

---

✅ 정리:  
`JSON_TABLE(p_cart_ids_json, '$[*]' COLUMNS (cid BIGINT PATH '$')) AS j`  
→ JSON 배열 파라미터를 **행(row) 집합으로 펼쳐서**, 각 원소를 `BIGINT` 타입 `cid` 컬럼으로 반환하고, 그 결과를 `j`라는 임시 테이블로 참조한다는 뜻입니다.