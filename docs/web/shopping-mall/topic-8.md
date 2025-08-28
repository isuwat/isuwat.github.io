---
layout: default
title: "[db] error logging"
parent: shop
grand_parent: Web
nav_order: 8
---

# 프로시저 에러 저장

### ✅ register_shop_cart (EXIT HANDLER 버전)
```sql
DELIMITER $$

DROP PROCEDURE IF EXISTS register_shop_cart $$
CREATE PROCEDURE register_shop_cart(
    IN  p_user_id    VARCHAR(20),
    IN  p_product_id VARCHAR(30),
    IN  p_qty        SMALLINT UNSIGNED,
    OUT po_result_code INT,
    OUT po_err_msg     VARCHAR(1024)
)
BEGIN
    -- ====== 메타/에러 변수 ======
    DECLARE v_proc_name VARCHAR(128) DEFAULT 'register_shop_cart';
    DECLARE v_proc_step INT UNSIGNED DEFAULT 0;
    DECLARE v_called_at DATETIME DEFAULT NOW();

    DECLARE v_sql_state VARCHAR(5);
    DECLARE v_err_no INT;
    DECLARE v_err_msg TEXT;
    DECLARE v_call_stack JSON;

    -- ====== 에러 핸들러 (모든 SQL 예외) ======
    DECLARE EXIT HANDLER FOR SQLEXCEPTION
    BEGIN
        -- 최근 에러 진단
        GET DIAGNOSTICS CONDITION 1
            v_sql_state = RETURNED_SQLSTATE,
            v_err_no    = MYSQL_ERRNO,
            v_err_msg   = MESSAGE_TEXT;

        ROLLBACK;

        -- OUT 파라미터 설정
        IF v_sql_state = '45000' THEN
            SET po_result_code = -100; -- 비즈니스 오류
            SET po_err_msg     = v_err_msg;
        ELSE
            SET po_result_code = -1;   -- 시스템 오류
            SET po_err_msg     = CONCAT('[', v_sql_state, '/', v_err_no, '] ', v_err_msg);
        END IF;

        -- 에러 로그 남기기 (insert_error 프로시저 호출 예시)
        CALL insert_error(
            v_proc_name,
            v_proc_step,
            CAST(v_call_stack AS CHAR),
            v_called_at,
            v_sql_state,
            po_result_code,
            po_err_msg,
            @dummy_result
        );
    END;

    -- ====== 초기화 ======
    SET po_result_code = 0;
    SET po_err_msg = NULL;
    SET v_call_stack = JSON_OBJECT(
        'p_user_id',    p_user_id,
        'p_product_id', p_product_id,
        'p_qty',        p_qty
    );

    -- ====== 트랜잭션 시작 ======
    SET v_proc_step = 10;
    START TRANSACTION;

    -- ====== 입력 검증 ======
    SET v_proc_step = 20;
    IF p_user_id IS NULL OR p_user_id = '' THEN
        SIGNAL SQLSTATE '45000' SET MESSAGE_TEXT = 'user_id is required';
    END IF;
    IF p_product_id IS NULL OR p_product_id = '' THEN
        SIGNAL SQLSTATE '45000' SET MESSAGE_TEXT = 'product_id is required';
    END IF;
    IF p_qty IS NULL OR p_qty <= 0 THEN
        SIGNAL SQLSTATE '45000' SET MESSAGE_TEXT = 'qty must be > 0';
    END IF;

    -- ====== 상품 유효성 검증 ======
    SET v_proc_step = 30;
    IF NOT EXISTS (
        SELECT 1 FROM v_products_effective v
         WHERE v.product_id = p_product_id
    ) THEN
        SIGNAL SQLSTATE '45000' SET MESSAGE_TEXT = 'Product is not purchasable';
    END IF;

    -- ====== 장바구니 등록/수량 누적 ======
    SET v_proc_step = 40;
    INSERT INTO shop_cart(user_id, product_id, quantity, create_date, update_date)
    VALUES (p_user_id, p_product_id, p_qty, NOW(), NOW())
    ON DUPLICATE KEY UPDATE
        quantity = quantity + VALUES(quantity),
        update_date = NOW();

    -- ====== 최대 수량 가드 ======
    SET v_proc_step = 50;
    UPDATE shop_cart
       SET quantity = LEAST(quantity, 65535) -- SMALLINT UNSIGNED 한계
     WHERE user_id = p_user_id
       AND product_id = p_product_id;

    -- ====== 성공 커밋 ======
    SET v_proc_step = 90;
    COMMIT;

    SET po_result_code = 0;
    SET po_err_msg = NULL;
END $$
DELIMITER ;

```

### 📌 동작 흐름

1. 에러 핸들러는 프로시저 시작부에서 선언 (DECLARE EXIT HANDLER FOR SQLEXCEPTION).

2. 내부에서 SIGNAL SQLSTATE '45000'을 사용해 비즈니스 오류를 명시적으로 발생시킴.

3. SQL 에러(제약 위반, FK 오류 등)는 핸들러에서 자동 잡힘.

4. 모든 오류는 insert_error 로깅 프로시저를 통해 기록.

5. 성공 시 po_result_code=0, 실패 시 음수 값 반환.