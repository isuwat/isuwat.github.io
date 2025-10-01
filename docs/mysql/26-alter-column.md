---
layout: default
title: "[MySQL] 컬럼 추가" 
parent: MySQL
nav_order: 26
---


### DDL (기간 없이 플래그)

```sql
ALTER TABLE shop_products
  ADD COLUMN is_best    TINYINT(1) NOT NULL DEFAULT 0 COMMENT '메인 BEST 노출',
  ADD COLUMN is_md_pick TINYINT(1) NOT NULL DEFAULT 0 COMMENT '메인 MD추천 노출',
  ADD COLUMN is_new     TINYINT(1) NOT NULL DEFAULT 0 COMMENT '메인 NEW 노출',

  ADD COLUMN best_rank SMALLINT UNSIGNED DEFAULT NULL COMMENT 'BEST 정렬(작을수록 상단)',
  ADD COLUMN md_rank   SMALLINT UNSIGNED DEFAULT NULL COMMENT 'MD추천 정렬',
  ADD COLUMN new_rank  SMALLINT UNSIGNED DEFAULT NULL COMMENT 'NEW 정렬';

-- 조회 최적화 인덱스
CREATE INDEX ix_products_best ON shop_products (is_best, best_rank, product_id);
CREATE INDEX ix_products_md   ON shop_products (is_md_pick, md_rank, product_id);
CREATE INDEX ix_products_new  ON shop_products (is_new, new_rank, product_id);
```
