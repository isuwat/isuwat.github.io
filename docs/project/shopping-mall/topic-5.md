---
layout: default
title: "[cart] view 테이블"
parent: shop
nav_order: 5
---

# 상품 VIEW 테이블 사용 이유

### 1️⃣ 원본 shop_products만으로는 “현재 판매 가능한 상품” 필터링이 불편

shop_products에는 상품 상태(stat), 노출여부(display), 재고수량(quantity_stock), 할인율(discount_rate), 최종가(final_price) 등이 흩어져 있습니다.  

매번 JOIN하거나 WHERE display='1' AND stat <> '2' AND quantity_stock > 0 같은 조건을 반복 작성해야 하죠.  

가독성이 떨어지고 유지보수가 어려워집니다.  

### 2️⃣ 뷰(View)로 “현재 판매 가능한 상품” 정의를 통합

v_products_effective 뷰는 보통 이런 식으로 정의합니다:  

```sql
CREATE OR REPLACE VIEW v_products_effective AS
SELECT 
    p.product_id,
    p.product_name,
    p.product_img_url AS image_url,
    COALESCE(p.final_price, p.sale_price) AS effective_price,
    p.display,
    p.stat,
    p.quantity_stock
FROM shop_products p
WHERE p.display = '1'       -- 노출 상품
  AND p.stat <> '2'         -- 품절 아님
  AND p.quantity_stock > 0; -- 재고 있음
```


👉 이렇게 해두면 장바구니, 주문, 검색 쿼리에서 항상 동일한 필터링 조건을 공유할 수 있습니다.  

### 3️⃣ 비즈니스 규칙을 “뷰”에서 캡슐화

할인율 적용 → final_price 계산 반영  

프로모션이나 시즌별 추가 조건 → 뷰 수정 한 번으로 전체 시스템 반영  

복잡한 조건식을 뷰에 숨기고, 프로시저는 “그냥 v_products_effective에서 가져온다”만 신경쓰면 됩니다.  

### 4️⃣ 장점 정리

일관성: 상품 유효성 체크가 어디서나 동일  

유지보수성: 규칙이 바뀌면 뷰만 수정 → 모든 프로시저 자동 반영  

안정성: 장바구니 담기 시점에서 “판매 가능 상품”이 아닌 경우 미리 차단  

가독성: 프로시저 코드가 단순해지고 SQL 조건이 짧아짐  

### 🔎 View의 특징

**1. 실제 데이터 없음**

CREATE VIEW v_products_effective AS SELECT ... FROM shop_products ...;  

이렇게 만들면, v_products_effective 자체는 행(row) 을 가지고 있지 않아요.  

실행할 때마다 shop_products (원본 테이블)에 가서 정의된 SELECT 문을 돌려서 결과를 보여줍니다.  

**2. 조회 시점마다 최신 데이터**  

예를 들어 shop_products에 display=0으로 바꾸거나 stat=2(품절)로 바꾸면,  

다음번 SELECT * FROM v_products_effective; 할 때 그 변경사항이 바로 반영됩니다.  

즉, 뷰는 항상 “원본 테이블의 현재 상태”를 보여줍니다.  

**3. 인덱스 없음**  

뷰에는 자체 인덱스가 없습니다.  

성능은 원본 테이블 인덱스에 달려 있어요.  

그래서 product_id, display, stat 같은 컬럼에 인덱스를 잘 잡아주는 게 중요합니다.  

**4. MERGE 가능 시 최적화**  

앞서 설명드린 ALGORITHM=MERGE가 가능하면, MySQL 옵티마이저가 뷰를 그냥 원본 쿼리로 인라인(inline)해 실행합니다.  

따라서 성능상 실제 테이블 조회와 큰 차이가 없습니다.  

✅ 결론:
v_products_effective 뷰는 **“현재 고객이 구매 가능한 상품 집합”**을 표준화한 계층입니다.  
그래서 장바구니, 주문, 재고 차감 로직 모두 이 뷰를 참조하면 중복 로직 방지 + 정책 변경 용이라는 장점이 있습니다.  