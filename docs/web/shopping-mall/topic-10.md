---
layout: default
title: "[db] 배송비 로직"
parent: shop
grand_parent: Web
nav_order: 10
---

# 장바구니 합계 금액을 보고 배송비를 자동 산정하는 로직

### 장바구니 합계(subtotal)에 따라 **배송비(shipping)**를 결정하는 조건 분기 로직입니다.

📌 코드 의미
```sql
IF subtotal = 0 OR subtotal >= shipping_free_threshold THEN
    SET shipping = 0;
ELSE
    SET shipping = shipping_flat;
END IF;
```


1. subtotal : 장바구니 총 상품 금액

2. shipping_free_threshold : 무료배송 기준 금액 (예: 50,000원)

3. shipping_flat : 기본 배송비 (예: 3,000원)

### 조건 해석

* subtotal = 0

    장바구니가 비었거나 금액이 0이면 → 배송비도 0

    (아예 주문할 게 없으니 배송비를 매길 필요가 없음)

* subtotal >= shipping_free_threshold

    장바구니 합계가 무료배송 기준 이상일 때 → 배송비 무료(0)

3. 그 외 (else)

    합계가 0도 아니고, 무료배송 기준에도 못 미치면 → shipping_flat (기본 배송비, 예: 3천원) 적용

### 📊 예시 상황

|subtotal (상품 합계)	|조건 판정	|shipping 결과|
|---|---|---|
|0|	subtotal = 0 → 무료	|0원|
|10,000	|기준 50,000 미만 → 기본 배송비	|3,000원|
|50,000	|무료배송 기준 충족	|0원|
|120,000	|무료배송 기준 초과	|0원|

### ✅ 요약

* 이 조건문은 장바구니 합계 금액을 보고 배송비를 자동 산정하는 로직입니다.

* 즉 “0원이거나 일정 금액 이상이면 무료배송, 아니면 기본 배송비” 규칙을 구현한 것입니다.

---

### 권장 그룹 구성
지금 등록하신 4개 배송정책에 대해 **묶음배송 그룹(combine_group_id)**을 이렇게 추천합니다.


1. 그룹 10: 일반 택배(동일 출고지/동일 택배사)

    - 포함 정책:

        1. 무조건 무료배송(free)

        2. 고정 3,000원(fixed)

        3. 3만원 이상 무료(conditional)

* 이유: 같은 물류센터/택배 라인에서 출고되는 “일반 상품군”이라면 정책이 달라도 같이 담아 1회 배송비 로직으로 계산하는 것이 자연스럽습니다.   
    (그룹 합계로 conditional 판단, free가 섞이면 그룹은 0원 등)

2. 그룹 20: 지역별/특수 정책 라인

    - 포함 정책:  
        4. 지역별(area)

        이유: 도서산간/권역별 가산·차등 등은 일반 라인과 묶음이 어렵고 계산 규칙도 다르므로 분리 권장.

### 바로 적용 SQL
```sql
UPDATE shop_delivery_policy SET combine_group_id = 10 WHERE id IN (1,2,3);
UPDATE shop_delivery_policy SET combine_group_id = 20 WHERE id IN (4);
```

### 계산 규칙(요약)

* 같은 그룹 내 묶음 1회만 부과

    free가 하나라도 있으면 → 그룹 배송비 0원

    conditional은 그룹 소계로 무료 여부 판단(예: 3만원 이상이면 0원, 미만이면 base_cost)

    fixed만 있으면 → 그룹당 1회 base_cost

* 그룹이 다르면 그룹별로 각각 위 규칙 적용 후 합산

### 팁

* 앞으로 출고지/택배사/상품 특성(냉장, 대형, 위험물) 이 분화되면

    일반택배-A센터: 10, 일반택배-B센터: 11, 냉장센터: 20, 화물: 30 …처럼 확장하세요.

* combine_group_id에 인덱스 걸어두면 장바구니 합계 계산시 그룹핑이 빨라집니다.
```sql
ALTER TABLE shop_delivery_policy ADD INDEX ix_combine_group (combine_group_id);
```