---
layout: default
title: "[option] 옵션 기능 정의"
parent: shop
nav_order: 27
---


# **메인 옵션(단일 셀렉트) 우선 구현** 기준

### 1) 테이블 DDL

```sql
/* 공통 옵션 */
SET NAMES utf8mb4;
SET FOREIGN_KEY_CHECKS = 0;

/* ─────────────────────────────────────────────
 * 상품 (이미 존재한다면 생략 가능)
 * ───────────────────────────────────────────── */
DROP TABLE IF EXISTS product;
CREATE TABLE product (
  product_id    BIGINT      NOT NULL AUTO_INCREMENT,
  name          VARCHAR(200) NOT NULL,
  base_price    INT          NOT NULL DEFAULT 0,
  is_active     TINYINT(1)   NOT NULL DEFAULT 1,
  PRIMARY KEY (product_id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

/* ─────────────────────────────────────────────
 * 조합형 SKU(메인 옵션용)
 * ───────────────────────────────────────────── */
DROP TABLE IF EXISTS product_variant;
CREATE TABLE product_variant (
  variant_id    BIGINT        NOT NULL AUTO_INCREMENT,
  product_id    BIGINT        NOT NULL,
  sku_code      VARCHAR(100)  NOT NULL,
  option_label  VARCHAR(200)  NOT NULL,   -- 셀렉트에 표시되는 완성형 라벨 (예: "네이비 / M")
  price         INT           NOT NULL,   -- 최종가(스냅샷형)
  stock_qty     INT           NOT NULL DEFAULT 0,
  is_active     TINYINT(1)    NOT NULL DEFAULT 1,
  img_url       VARCHAR(500)  NULL,
  updated_at    TIMESTAMP     NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  PRIMARY KEY (variant_id),
  UNIQUE KEY uk_variant_sku (sku_code),
  KEY idx_variant_product (product_id),
  CONSTRAINT fk_variant_product FOREIGN KEY (product_id) REFERENCES product(product_id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

/* ─────────────────────────────────────────────
 * 장바구니 (메인 옵션 우선) — 애드온 확장 대비 group_id 포함
 * ───────────────────────────────────────────── */
DROP TABLE IF EXISTS cart_group;
CREATE TABLE cart_group (         -- 묶음 식별(추후 애드온 용이)
  group_id     BIGINT NOT NULL AUTO_INCREMENT,
  created_at   DATETIME NOT NULL DEFAULT NOW(),
  PRIMARY KEY (group_id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

DROP TABLE IF EXISTS shop_cart;
CREATE TABLE shop_cart (
  cart_id       BIGINT      NOT NULL AUTO_INCREMENT,
  user_id       BIGINT      NOT NULL,
  product_id    BIGINT      NOT NULL,
  variant_id    BIGINT      NOT NULL,
  sku_code      VARCHAR(100) NOT NULL,    -- 스냅샷
  option_text   VARCHAR(200) NOT NULL,    -- 스냅샷(표시용 라벨)
  unit_price    INT         NOT NULL,     -- 스냅샷(가격)
  qty           INT         NOT NULL,
  group_id      BIGINT      NULL,         -- 메인/애드온 묶음 (메인만이면 NULL 가능)
  created_at    DATETIME    NOT NULL DEFAULT NOW(),
  deleted_at    DATETIME    NULL,         -- 소프트 삭제
  PRIMARY KEY (cart_id),
  KEY idx_cart_user (user_id),
  KEY idx_cart_group (group_id),
  CONSTRAINT fk_cart_variant FOREIGN KEY (variant_id) REFERENCES product_variant(variant_id),
  CONSTRAINT fk_cart_product FOREIGN KEY (product_id) REFERENCES product(product_id),
  CONSTRAINT fk_cart_group   FOREIGN KEY (group_id)   REFERENCES cart_group(group_id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

/* ─────────────────────────────────────────────
 * 주문 헤더/아이템 (간단 스키마; 기존 테이블 있으면 맞춰 사용)
 * ───────────────────────────────────────────── */
DROP TABLE IF EXISTS shop_order_items;
DROP TABLE IF EXISTS shop_orders;

CREATE TABLE shop_orders (
  order_id       BIGINT       NOT NULL AUTO_INCREMENT,
  order_number   VARCHAR(40)  NOT NULL,   -- 고유 주문번호(yyyyMMddhhmmss + seq 등)
  user_id        BIGINT       NOT NULL,
  order_date     DATETIME     NOT NULL DEFAULT NOW(),
  order_status   VARCHAR(40)  NOT NULL DEFAULT 'shipped_before', -- 예: 입금전 등
  total_payment  INT          NOT NULL DEFAULT 0,
  PRIMARY KEY (order_id),
  UNIQUE KEY uk_order_number (order_number),
  KEY idx_order_user (user_id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

CREATE TABLE shop_order_items (
  order_item_id  BIGINT       NOT NULL AUTO_INCREMENT,
  order_id       BIGINT       NOT NULL,
  product_id     BIGINT       NOT NULL,
  variant_id     BIGINT       NOT NULL,
  sku_code       VARCHAR(100) NOT NULL,   -- 스냅샷
  option_text    VARCHAR(200) NOT NULL,   -- 스냅샷
  unit_price     INT          NOT NULL,   -- 스냅샷
  qty            INT          NOT NULL,
  line_total     INT          NOT NULL,   -- unit_price * qty
  PRIMARY KEY (order_item_id),
  KEY idx_item_order (order_id),
  CONSTRAINT fk_item_order   FOREIGN KEY (order_id)   REFERENCES shop_orders(order_id),
  CONSTRAINT fk_item_variant FOREIGN KEY (variant_id) REFERENCES product_variant(variant_id),
  CONSTRAINT fk_item_product FOREIGN KEY (product_id) REFERENCES product(product_id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

SET FOREIGN_KEY_CHECKS = 1;

```

# 2) 저장 프로시저 (SP)

> MySQL 8.0, `DELIMITER` 사용. 오류 시 ROLLBACK, 성공 시 COMMIT.  
> PHP(PDO)에서 `CALL ...` 후 OUT 변수는 `SELECT @rc, @msg` 패턴으로 조회.

```sql
DELIMITER $$

/* ─────────────────────────────────────────────
 * 1) 메인 옵션 목록: 상세 페이지 렌더용
 * ───────────────────────────────────────────── */
DROP PROCEDURE IF EXISTS sp_get_product_variants $$
CREATE PROCEDURE sp_get_product_variants(IN pi_product_id BIGINT)
BEGIN
  SELECT
    v.variant_id,
    v.product_id,
    v.sku_code,
    v.option_label,
    v.price,
    v.stock_qty,
    v.is_active,
    v.img_url
  FROM product_variant v
  WHERE v.product_id = pi_product_id
  ORDER BY v.option_label ASC;
END $$

/* ─────────────────────────────────────────────
 * 2) SKU로 상세 조회: 선택 시 가격/재고/이미지 갱신
 * ───────────────────────────────────────────── */
DROP PROCEDURE IF EXISTS sp_get_variant_by_sku $$
CREATE PROCEDURE sp_get_variant_by_sku(IN pi_sku_code VARCHAR(100))
BEGIN
  SELECT
    v.variant_id,
    v.product_id,
    v.sku_code,
    v.option_label,
    v.price,
    v.stock_qty,
    v.is_active,
    v.img_url
  FROM product_variant v
  WHERE v.sku_code = pi_sku_code
  LIMIT 1;
END $$

/* ─────────────────────────────────────────────
 * 3) 장바구니 담기(메인만) — 검증 + 스냅샷 저장
 *    po_rc: 0=성공, 400/409 등 오류코드
 * ───────────────────────────────────────────── */
DROP PROCEDURE IF EXISTS sp_cart_add $$
CREATE PROCEDURE sp_cart_add(
  IN  pi_user_id     BIGINT,
  IN  pi_product_id  BIGINT,
  IN  pi_variant_id  BIGINT,
  IN  pi_qty         INT,
  OUT po_rc          INT,
  OUT po_msg         VARCHAR(200)
)
proc:BEGIN
  DECLARE v_price INT; DECLARE v_stock INT; DECLARE v_active TINYINT;
  DECLARE v_pid BIGINT; DECLARE v_sku VARCHAR(100); DECLARE v_label VARCHAR(200);

  IF pi_qty IS NULL OR pi_qty <= 0 THEN SET po_rc=400; SET po_msg='Invalid qty'; LEAVE proc; END IF;

  SELECT product_id, price, stock_qty, is_active, sku_code, option_label
    INTO v_pid,     v_price, v_stock,     v_active, v_sku,   v_label
  FROM product_variant
  WHERE variant_id = pi_variant_id
  LIMIT 1;

  IF v_pid IS NULL OR v_pid <> pi_product_id THEN SET po_rc=400; SET po_msg='Invalid variant'; LEAVE proc; END IF;
  IF v_active = 0 THEN SET po_rc=400; SET po_msg='Inactive variant'; LEAVE proc; END IF;
  IF v_stock < pi_qty THEN SET po_rc=409; SET po_msg='Out of stock'; LEAVE proc; END IF;

  INSERT INTO shop_cart (user_id, product_id, variant_id, sku_code, option_text, unit_price, qty, created_at)
  VALUES (pi_user_id, pi_product_id, pi_variant_id, v_sku, v_label, v_price, pi_qty, NOW());

  SET po_rc=0; SET po_msg='OK';
END $$

/* ─────────────────────────────────────────────
 * 4) 주문 생성(카트 선택행 → 주문행)
 *    - 입력: 유저, 카트ID JSON: [1,2,3]
 *    - 정책: 성공시에만 재고 차감 및 카트 소프트 삭제
 * ───────────────────────────────────────────── */
DROP PROCEDURE IF EXISTS sp_order_create_from_cart $$
CREATE PROCEDURE sp_order_create_from_cart(
  IN  pi_user_id        BIGINT,
  IN  pi_cart_ids_json  JSON,
  OUT po_rc             INT,
  OUT po_msg            VARCHAR(200)
)
proc:BEGIN
  DECLARE v_order_id BIGINT;
  DECLARE v_total INT DEFAULT 0;

  DECLARE EXIT HANDLER FOR SQLEXCEPTION
  BEGIN
    ROLLBACK;
    SET po_rc=500; SET po_msg='DB Error';
  END;

  START TRANSACTION;

  /* 4.1 선택 카트행 잠금 + 유효성 검증 */
  WITH selected_ids AS (
    SELECT CAST(j.cid AS UNSIGNED) AS cart_id
    FROM JSON_TABLE(pi_cart_ids_json, '$[*]' COLUMNS (cid VARCHAR(20) PATH '$')) AS j
  ),
  cart_sel AS (
    SELECT c.cart_id, c.product_id, c.variant_id, c.sku_code, c.option_text, c.unit_price, c.qty
    FROM shop_cart c
    JOIN selected_ids s ON s.cart_id = c.cart_id
    WHERE c.user_id = pi_user_id AND c.deleted_at IS NULL
    FOR UPDATE
  ),
  vcheck AS (
    SELECT v.variant_id, v.stock_qty, v.is_active
    FROM product_variant v
    JOIN cart_sel c ON c.variant_id = v.variant_id
    FOR UPDATE
  )
  SELECT 0 INTO po_rc;

  /* 재고/활성 체크 */
  IF EXISTS (
    SELECT 1 FROM product_variant v
    JOIN cart_sel c ON c.variant_id = v.variant_id
    WHERE v.is_active = 0 OR v.stock_qty < c.qty
  ) THEN
    SET po_rc=409; SET po_msg='Out of stock or inactive during order';
    ROLLBACK; LEAVE proc;
  END IF;

  /* 4.2 주문 헤더 생성 */
  INSERT INTO shop_orders (order_number, user_id, order_date, order_status, total_payment)
  VALUES (DATE_FORMAT(NOW(), '%Y%m%d%H%i%s'), pi_user_id, NOW(), 'shipped_before', 0);
  SET v_order_id = LAST_INSERT_ID();

  /* 4.3 주문 아이템 생성 + 합계 */
  INSERT INTO shop_order_items (order_id, product_id, variant_id, sku_code, option_text, unit_price, qty, line_total)
  SELECT v_order_id, c.product_id, c.variant_id, c.sku_code, c.option_text, c.unit_price, c.qty, (c.unit_price * c.qty)
  FROM cart_sel c;

  SELECT COALESCE(SUM(line_total),0) INTO v_total
  FROM shop_order_items WHERE order_id = v_order_id;

  UPDATE shop_orders SET total_payment = v_total WHERE order_id = v_order_id;

  /* 4.4 재고 차감 */
  UPDATE product_variant v
  JOIN cart_sel c ON c.variant_id = v.variant_id
  SET v.stock_qty = v.stock_qty - c.qty;

  /* 4.5 사용된 카트 소프트 삭제 */
  UPDATE shop_cart c
  JOIN (SELECT cart_id FROM JSON_TABLE(pi_cart_ids_json, '$[*]' COLUMNS (cid VARCHAR(20) PATH '$')) j) s
    ON c.cart_id = CAST(s.cid AS UNSIGNED)
  SET c.deleted_at = NOW()
  WHERE c.user_id = pi_user_id AND c.deleted_at IS NULL;

  COMMIT;
  SET po_rc=0; SET po_msg='OK';
END $$

DELIMITER ;

```


## 3) 호출 예시 (PHP/PDO)

- **옵션 목록**: `CALL sp_get_product_variants(:pid)` → `fetchAll()`
    
- **SKU 상세**: `CALL sp_get_variant_by_sku(:sku)` → `fetch()`
    
- **장바구니 담기**:

```php
$dbh->query("SET @rc := -1; SET @msg := ''");
$stmt = $dbh->prepare("CALL sp_cart_add(:uid,:pid,:vid,:qty,@rc,@msg)");
$stmt->execute([':uid'=>$userId, ':pid'=>$productId, ':vid'=>$variantId, ':qty'=>$qty]);
$res = $dbh->query("SELECT @rc rc, @msg msg")->fetch(PDO::FETCH_ASSOC);

```

* **주문 생성**:
```php
$dbh->query("SET @rc := -1; SET @msg := ''");
$stmt = $dbh->prepare("CALL sp_order_create_from_cart(:uid, :json, @rc, @msg)");
$stmt->execute([':uid'=>$userId, ':json'=>json_encode($cartIds)]);
$res = $dbh->query("SELECT @rc rc, @msg msg")->fetch(PDO::FETCH_ASSOC);

```

### 메모

- 이미 `product`, `shop_orders`, `shop_order_items`가 있다면 **해당 테이블명/컬럼**에 맞춰 SP의 `INSERT/SELECT` 부분만 맞추면 됩니다.
    
- 애드온 확장 시에는 `cart_group`/`shop_cart.group_id`를 활용해 **메인+애드온 묶음**을 표기하고, `sp_cart_add_with_addons`를 추가로 정의하면 됩니다(메인 구조는 그대로 재사용).