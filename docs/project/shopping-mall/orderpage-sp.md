---
layout: default
title: "[cart] 주문페이지 procedure"
parent: shop
nav_order: 24
---



# 주문페이지 procedure

- 중간 계산·제어용은 모두 `DECLARE` 로컬 변수로 처리
    
- 외부(PHP)에서 `SELECT @rc, @msg`로 재확인하고 싶을 때를 대비해 마지막에 `@rc`, `@msg` 세션 변수도 세팅
    
- 1137 회피: 합계/배송비 계산을 **여러 문장으로 분리**하여 임시테이블 재오픈 방지
    
- 예외 시에도 임시테이블 정리 보장

```sql
DELIMITER $$

CREATE PROCEDURE `shop_cart_selected_order_form`(
    IN  pi_user_id       VARCHAR(20),
    IN  pi_cart_ids      JSON,
    IN  pi_scope         VARCHAR(20),     -- 'all' | 'selected'
    OUT po_result_code   INT,
    OUT po_err_msg       VARCHAR(255)
)
BEGIN
    /****************************************************
     * Procedure : shop_cart_selected_order_form
     * 작성자     : isuwat
     * 최종수정   : 2025-09-07
     * 설명       : 유저정보 + 장바구니 선택목록 + 합계/배송비
     ****************************************************/

    /* =========================
     * 로컬 변수(DECLARE) : 프로시저 내부 제어/계산 전용
     * ========================= */
    DECLARE v_proc_name   VARCHAR(128) DEFAULT 'shop_cart_selected_order_form';
    DECLARE v_proc_step   INT UNSIGNED DEFAULT 0;
    DECLARE v_called_at   DATETIME     DEFAULT NOW();
    DECLARE v_sql_state   VARCHAR(5);
    DECLARE v_err_no      INT;
    DECLARE v_err_msg_t   TEXT;
    DECLARE v_call_stack  JSON;

    DECLARE v_scope       VARCHAR(20);
    DECLARE v_subtotal    BIGINT UNSIGNED DEFAULT 0;
    DECLARE v_shipping    BIGINT UNSIGNED DEFAULT 0;
    DECLARE v_discount    BIGINT UNSIGNED DEFAULT 0; -- 확장 대비
    DECLARE v_total       BIGINT UNSIGNED DEFAULT 0;

    /* =========================
     * 세션 변수(@) : 외부에서 조회/공유 목적
     *  - @log_ok : insert_error용 세션 플래그(타 프로시저와 공유)
     *  - @rc, @msg : 외부(PHP)에서 재확인 편의
     * ========================= */
    SET @log_ok := NULL;
    SET @rc := NULL;
    SET @msg := NULL;

    /* 예외 핸들러 */
    DECLARE EXIT HANDLER FOR SQLEXCEPTION
    BEGIN
        /* 에러 정보 수집 */
        GET DIAGNOSTICS CONDITION 1
            v_sql_state = RETURNED_SQLSTATE,
            v_err_no    = MYSQL_ERRNO,
            v_err_msg_t = MESSAGE_TEXT;

        /* 임시테이블 정리 */
        DROP TEMPORARY TABLE IF EXISTS tmp_selected_ids;
        DROP TEMPORARY TABLE IF EXISTS tmp_cart_items;

        /* 롤백 */
        ROLLBACK;

        /* 사용자 정의 에러(45000) vs 일반 에러 */
        IF v_sql_state = '45000' THEN
            SET po_result_code = -100;
            SET po_err_msg     = v_err_msg_t;
        ELSE
            SET po_result_code = -1;
            SET po_err_msg     = CONCAT('[', v_sql_state, '/', v_err_no, '] ', v_err_msg_t);
        END IF;

        /* 로그 기록 (세션 변수 @log_ok 사용) */
        SET v_call_stack = JSON_OBJECT('pi_user_id', pi_user_id, 'pi_cart_ids', pi_cart_ids, 'pi_scope', pi_scope);
        CALL insert_error(
            v_proc_name, v_proc_step, JSON_UNQUOTE(v_call_stack),
            v_called_at, v_sql_state, po_result_code, po_err_msg, @log_ok
        );

        /* 외부 조회 편의값 */
        SET @rc  = po_result_code;
        SET @msg = po_err_msg;
    END;

    /* 초기화 */
    SET po_result_code = 0;
    SET po_err_msg     = NULL;
    SET v_call_stack   = JSON_OBJECT('pi_user_id', pi_user_id, 'pi_cart_ids', pi_cart_ids, 'pi_scope', pi_scope);

    /* 입력값 검증 */
    SET v_proc_step = 10;
    IF pi_user_id IS NULL OR pi_user_id = '' THEN
        SIGNAL SQLSTATE '45000' SET MESSAGE_TEXT = 'user_id is required';
    END IF;

    IF pi_cart_ids IS NULL OR JSON_TYPE(pi_cart_ids) <> 'ARRAY' OR JSON_LENGTH(pi_cart_ids) = 0 THEN
        SIGNAL SQLSTATE '45000' SET MESSAGE_TEXT = 'selected_cart_ids json array is required';
    END IF;

    SET v_scope = NULLIF(TRIM(pi_scope), '');

    /* =========================
     * 임시테이블1: 선택된 cart_id
     * ========================= */
    SET v_proc_step = 20;
    DROP TEMPORARY TABLE IF EXISTS tmp_selected_ids;
    CREATE TEMPORARY TABLE tmp_selected_ids (
        cart_id INT UNSIGNED PRIMARY KEY
    ) ENGINE=InnoDB;

    IF v_scope = 'selected' THEN
        INSERT INTO tmp_selected_ids(cart_id)
        SELECT CAST(j.cid AS UNSIGNED)
        FROM JSON_TABLE(pi_cart_ids, '$[*]' COLUMNS (cid INT PATH '$')) AS j;
    ELSEIF v_scope = 'all' THEN
        INSERT INTO tmp_selected_ids(cart_id)
        SELECT seq_no
        FROM shop_cart
        WHERE user_id = pi_user_id;
    ELSE
        SIGNAL SQLSTATE '45000' SET MESSAGE_TEXT = '잘못된 주문 요청입니다.';
    END IF;

    /* =========================
     * 임시테이블2: 장바구니 상세 스냅샷
     * ========================= */
    SET v_proc_step = 30;
    DROP TEMPORARY TABLE IF EXISTS tmp_cart_items;
    CREATE TEMPORARY TABLE tmp_cart_items (
        cart_id               INT UNSIGNED,
        category_id           INT UNSIGNED,
        product_id            VARCHAR(30),
        product_name          TEXT,
        sale_price            BIGINT UNSIGNED,
        final_price           BIGINT UNSIGNED,
        quantity              INT,
        row_sum               BIGINT UNSIGNED,
        dis_rate              INT UNSIGNED,
        product_thumb_img_url VARCHAR(200),
        delivery_policy_id    INT,
        type                  VARCHAR(32),
        base_cost             INT UNSIGNED,
        free_over             INT UNSIGNED,
        combine_group_id      INT
        ,PRIMARY KEY(cart_id)
        ,KEY idx_prod(product_id)
        ,KEY idx_policy(delivery_policy_id, combine_group_id)
    ) ENGINE=InnoDB;

    INSERT INTO tmp_cart_items (
        cart_id, category_id, product_id, product_name, sale_price,
        final_price, quantity, row_sum, dis_rate, product_thumb_img_url,
        delivery_policy_id, type, base_cost, free_over, combine_group_id
    )
    SELECT
        c.seq_no,
        p.category_id,
        c.product_id,
        v.product_name,
        v.sale_price,
        v.final_price,
        c.quantity,
        (v.final_price * c.quantity) AS row_sum,
        CAST(p.discount_rate * 100 AS UNSIGNED) AS dis_rate,
        v.image_url,
        p.delivery_policy_id,
        dp.type,
        dp.base_cost,
        dp.free_over,
        COALESCE(dp.combine_group_id, -1) AS combine_group_id
    FROM shop_cart c
    JOIN tmp_selected_ids s           ON s.cart_id = c.seq_no
    JOIN v_products_effective v       ON v.product_id = c.product_id
    JOIN shop_products p              ON p.product_id = c.product_id
    LEFT JOIN shop_delivery_policy dp ON dp.id = p.delivery_policy_id
    WHERE c.user_id = pi_user_id;

    /* =========================
     * 결과셋 #1: 회원/기본배송지
     * ========================= */
    SET v_proc_step = 40;
    SELECT 
        m.cust_no,
        m.user_cash,
        ssa.receiver_name,
        ssa.receiver_phone,
        ssa.post_code,
        ssa.address,
        ssa.address_detail,
        ssa.delivery_msg
    FROM member AS m
    LEFT JOIN (
        SELECT s.user_id,
               s.receiver_name,
               s.receiver_phone,
               s.post_code,
               s.address,
               s.address_detail,
               s.delivery_msg
        FROM shop_shipping_address s
        WHERE s.user_id = pi_user_id
        ORDER BY s.is_default DESC, s.update_date DESC
        LIMIT 1
    ) AS ssa
      ON ssa.user_id = m.user_id
    WHERE m.user_id = pi_user_id;

    /* =========================
     * 결과셋 #2: 아이템 목록
     * ========================= */
    SET v_proc_step = 50;
    SELECT
        cart_id,
        category_id,
        product_id,
        product_name,
        sale_price,
        final_price,
        quantity,
        row_sum,
        dis_rate,
        product_thumb_img_url
    FROM tmp_cart_items
    ORDER BY cart_id;

    /* =========================
     * 결과셋 #3: 합계/배송비 (1137 회피: 문장 분리)
     * ========================= */
    SET v_proc_step = 60;

    /* 1) 소계 */
    SELECT COALESCE(SUM(row_sum), 0) INTO v_subtotal
    FROM tmp_cart_items;

    /* 2) 배송비 */
    SELECT
        COALESCE(SUM(
            CASE
                WHEN type = 'free'        THEN 0
                WHEN type = 'fixed'       THEN base_cost
                WHEN type = 'conditional' THEN
                    CASE
                        WHEN group_subtotal = 0 THEN 0
                        WHEN free_over IS NOT NULL AND group_subtotal >= free_over THEN 0
                        ELSE base_cost
                    END
                WHEN type = 'area'        THEN base_cost      -- TODO: 지역별 상세 로직
                ELSE 0
            END
        ), 0) AS shipping_total
    INTO v_shipping
    FROM (
        SELECT
            delivery_policy_id,
            type,
            base_cost,
            free_over,
            COALESCE(combine_group_id, -1) AS combine_group_id,
            SUM(final_price * quantity) AS group_subtotal
        FROM tmp_cart_items
        GROUP BY delivery_policy_id, type, base_cost, free_over, combine_group_id
    ) g;

    /* 3) 할인(추후 확장) & 총계 */
    SET v_discount := 0;
    SET v_total    := v_subtotal + v_shipping - v_discount;

    /* 최종 한 줄 결과셋 */
    SELECT
        v_subtotal AS subtotal,
        v_shipping AS shipping,
        v_discount AS discount,
        v_total    AS total;

    /* 정리 */
    DROP TEMPORARY TABLE IF EXISTS tmp_selected_ids;
    DROP TEMPORARY TABLE IF EXISTS tmp_cart_items;

    /* OUT 파라미터 & 세션 변수 결과 */
    SET po_result_code = 0;
    SET po_err_msg     = 'All operations successful';

    SET @rc  = po_result_code;  -- 외부(PHP)에서 SELECT @rc, @msg 편의
    SET @msg = po_err_msg;
END$$

DELIMITER ;

```

### 왜 이렇게 나눴나 (요점)

- **`DECLARE` 로컬 변수**: `v_subtotal`, `v_shipping`, `v_total` 등 프로시저 내부 계산/제어용. 블록이 끝나면 자동 소멸.
    
- **`@세션 변수`**: `@rc`, `@msg`, `@log_ok`처럼 **프로시저 바깥(같은 세션)**에서도 조회·공유할 값.
    
- **1137 회피**: `tmp_cart_items`를 한 **SELECT 문에서 2회 이상** 열지 않도록 **문장 분리**로 구현.