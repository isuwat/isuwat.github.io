---
layout: default
title: "[admin] BEST MD NEW 설정 UI 추가"
parent: admin
nav_order: 8
---


**Summary**  
신규 상품 등록 팝업에 메인 섹션 노출 설정(BEST/MD추천/NEW)과 정렬순서를 입력할 수 있는 UI를 추가하고, 저장 요청 페이로드에 반영한다.

**Parent**  
Story: 홈 섹션 노출 v1 (BEST/MD/NEW; 기간 없음)

## 배경/목표

메인 페이지의 BEST/MD추천/NEW 섹션 운영을 위해 상품 등록 시점에 섹션 노출 여부와 정렬순서를 지정할 수 있어야 한다. 기간 제어는 포함하지 않으며, 단순 플래그+정렬만 지원한다.

---

## 범위 (Scope)

- 파일: `products/register_shop_product_popup.php`    
- UI 요소 추가    
    - 체크박스: `is_best`, `is_md_pick`, `is_new`        
    - 숫자 입력: `best_rank`, `md_rank`, `new_rank` (NULL 허용)        
- 클라이언트 검증    
    - 체크박스는 0/1로 변환        
    - 정렬값은 빈값 허용, 입력 시 0 이상 정수(권장 범위: 0~32767)        
- 전송 페이로드 확장 (AJAX JSON POST)    
    - 키: `is_best`, `is_md_pick`, `is_new`, `best_rank`, `md_rank`, `new_rank`        
    - 기존 필드와 동일한 전송 방식 유지        
- UX    
    - 각 필드 라벨/헬프텍스트 추가        
    - 폼 저장 시 비활성화 처리, 저장 성공/실패 토스트/모달 기존 방식 유지        

> **비고**: DB DDL, 인덱스, 서버/프로시저 파라미터 추가는 별도 태스크(S1, S4)에서 수행.

## Acceptance Criteria (AC)

1. **UI 표시**: 등록 팝업에 BEST/MD추천/NEW 체크박스 및 각 정렬 입력이 노출된다.    
2. **검증 규칙**    
    - 정렬 입력이 비어 있으면 `NULL`로 취급(전송 생략 또는 `null`).        
    - 정렬 입력 시 0 이상의 정수만 허용(음수/소수/문자 입력 차단).        
3. **페이로드 반영**    
    - 체크박스 선택 시 `1`, 미선택 시 `0`으로 전송.        
    - 정렬 입력은 공백→미전송(또는 `null`), 숫자 입력 시 정수로 전송.        
4. **서버 연계 호환**    
    - 서버(프로시저)에서 파라미터 미전달 또는 `null`/`0`일 때 기본값으로 안전 저장.     
    - 기존 필드 전송/동작에 영향 없음.        
5. **UX/회귀**    
    - 저장 성공 시 기존 성공 플로우(알림 후 닫기/리스트 갱신) 유지.        
    - 기존 필수값 미입력 에러 시 현재와 동일 패턴의 에러 노출.

## API 계약(프런트 → 서버)

- `is_best` (0|1, number)    
- `is_md_pick` (0|1, number)    
- `is_new` (0|1, number)    
- `best_rank` (nullable integer ≥0)    
- `md_rank` (nullable integer ≥0)    
- `new_rank` (nullable integer ≥0)    

> 서버/프로시저 파라미터명 권장:  
> `pi_is_best, pi_best_rank, pi_is_md_pick, pi_md_rank, pi_is_new, pi_new_rank`  
> (S4에서 실제 명세 확정)


### 1) JS 페이로드 구성 — 체크박스/정렬 값을 `data`에 반영

```js
// 메인 노출 값 수집
const is_best    = $('#is_best').is(':checked') ? 1 : 0;
const is_md_pick = $('#is_md_pick').is(':checked') ? 1 : 0;
const is_new     = $('#is_new').is(':checked') ? 1 : 0;

// 정렬값: 비어있으면 전달 생략(=서버에서 NULL 처리)
const best_rank_raw = $('#best_rank').val().trim();
const md_rank_raw   = $('#md_rank').val().trim();
const new_rank_raw  = $('#new_rank').val().trim();

const best_rank = best_rank_raw === '' ? undefined : parseInt(best_rank_raw, 10);
const md_rank   = md_rank_raw   === '' ? undefined : parseInt(md_rank_raw,   10);
const new_rank  = new_rank_raw  === '' ? undefined : parseInt(new_rank_raw,  10);

```

```js
const data = {
  // ...기존 필드들...
  origin: $('#origin').val().trim(),

  // 메인 노출 플래그
  is_best:    is_best,
  is_md_pick: is_md_pick,
  is_new:     is_new,
};

```

**`const data = { ... }` 생성 직후**, 정렬값이 있을 때만 키를 추가)

```js
if (best_rank !== undefined) data.best_rank = best_rank;
if (md_rank   !== undefined) data.md_rank   = md_rank;
if (new_rank  !== undefined) data.new_rank  = new_rank;

```

> 참고: 이 파일은 `$.ajax({ data: JSON.stringify(data) })`로 **JSON 문자열**을 보냅니다. `undefined` 속성은 `JSON.stringify`에서 **자동으로 제외**되어 비어 있으면 아예 전송되지 않습니다(서버에서 기본값/NULL 처리에 유리).

---

## 동작 확인 체크리스트 (빠른 수동 테스트)

1. 체크 OFF + 정렬 공백 → 요청 JSON에 `is_*:0`만 포함, `*_rank` 없음.    
2. BEST만 체크, 정렬 공백 → `is_best:1`, `best_rank` 없음.    
3. BEST/MD/NEW 모두 체크 + 정렬 각각 0/10/999 → 세 플래그와 세 정렬 숫자 포함.    
4. 정렬에 음수/문자 입력 → 기존 숫자 검증 로직이 경고 후 포커스 이동.    
5. 기존 필수값/가격 검증/이미지 URL 검증 플로우 **변화 없음**.


### 2) 프로시저 시그니처 확장

```sql
DROP PROCEDURE IF EXISTS register_adm_shop_product;
DELIMITER $$

CREATE PROCEDURE register_adm_shop_product(
    -- [기존 파라미터들...]
    -- 예: IN pi_product_name VARCHAR(200), IN pi_sale_price BIGINT UNSIGNED, ...

    -- [추가 파라미터: 홈 섹션]
    IN  pi_is_best    TINYINT,           -- 0/1 (NULL 허용 → 0 처리)
    IN  pi_best_rank  INT,               -- NULL 허용, 0 이상
    IN  pi_is_md_pick TINYINT,
    IN  pi_md_rank    INT,
    IN  pi_is_new     TINYINT,
    IN  pi_new_rank   INT,

    OUT po_result_code INT,
    OUT po_err_msg     VARCHAR(255)
)
BEGIN
    DECLARE v_sql_state CHAR(5) DEFAULT '00000';
    DECLARE v_err_no INT DEFAULT 0;
    DECLARE v_err_msg VARCHAR(255) DEFAULT '';

    -- 정규화된 값 보관 변수
    DECLARE v_is_best    TINYINT   DEFAULT 0;
    DECLARE v_is_md_pick TINYINT   DEFAULT 0;
    DECLARE v_is_new     TINYINT   DEFAULT 0;

    DECLARE v_best_rank  SMALLINT UNSIGNED DEFAULT NULL;
    DECLARE v_md_rank    SMALLINT UNSIGNED DEFAULT NULL;
    DECLARE v_new_rank   SMALLINT UNSIGNED DEFAULT NULL;

    DECLARE EXIT HANDLER FOR SQLEXCEPTION
    BEGIN
        GET DIAGNOSTICS CONDITION 1
            v_sql_state = RETURNED_SQLSTATE,
            v_err_no    = MYSQL_ERRNO,
            v_err_msg   = MESSAGE_TEXT;
        ROLLBACK;
        SET po_result_code = v_err_no;
        SET po_err_msg     = CONCAT('register_adm_shop_product error: ', v_err_msg);
    END;

    START TRANSACTION;

    /* 입력 정규화: NULL → 0, 비0 → 1 */
    SET v_is_best    = (IFNULL(pi_is_best,    0) <> 0);
    SET v_is_md_pick = (IFNULL(pi_is_md_pick, 0) <> 0);
    SET v_is_new     = (IFNULL(pi_is_new,     0) <> 0);

    /* 랭크: NULL 허용, 음수는 NULL, 상한 보호(32767) */
    SET v_best_rank = CASE
        WHEN pi_best_rank IS NULL OR pi_best_rank < 0 THEN NULL
        WHEN pi_best_rank > 32767 THEN 32767
        ELSE CAST(pi_best_rank AS UNSIGNED)
    END;

    SET v_md_rank = CASE
        WHEN pi_md_rank IS NULL OR pi_md_rank < 0 THEN NULL
        WHEN pi_md_rank > 32767 THEN 32767
        ELSE CAST(pi_md_rank AS UNSIGNED)
    END;

    SET v_new_rank = CASE
        WHEN pi_new_rank IS NULL OR pi_new_rank < 0 THEN NULL
        WHEN pi_new_rank > 32767 THEN 32767
        ELSE CAST(pi_new_rank AS UNSIGNED)
    END;

    /* 필수값 검증(예시) — 필요 시 기존 규칙과 동일하게 확장 */
    -- IF (pi_product_name IS NULL OR pi_product_name = '') THEN
    --     SIGNAL SQLSTATE '45000' SET MESSAGE_TEXT = 'product_name is required';
    -- END IF;

    /* INSERT: 기존 컬럼 리스트에 아래 6개를 추가 */
    INSERT INTO shop_products (
        /* [기존 칼럼들 …] */
        is_best, best_rank,
        is_md_pick, md_rank,
        is_new, new_rank
    ) VALUES (
        /* [기존 VALUES …] */
        v_is_best, v_best_rank,
        v_is_md_pick, v_md_rank,
        v_is_new, v_new_rank
    );

    COMMIT;

    SET po_result_code = 0;
    SET po_err_msg     = NULL;
END$$
DELIMITER ;

```

### 3) 서버(PHP) 연동 메모

- `exec_register_adm_shop_product.php`에서 들어오는 JSON 키를 그대로 바인딩:    
    - `is_best, best_rank, is_md_pick, md_rank, is_new, new_rank`        
- **미전달/공백** → `NULL` 바인딩 권장(정렬은 NULL로 저장되며 하단 정렬)    
- 플래그는 `0/1`로 바인딩(문자 `"1"`도 OK)    
- 오류 시 `po_result_code != 0`이면 메시지 반환 UX 기존과 동일