---
layout: default
title: "[cart] 옵션"
parent: shop
grand_parent: Web
nav_order: 21
---


# 장바구니 옵션
## 1) 카트 row의 유니크 기준

- `UNIQUE (user_id, product_id, option_hash)`
    
    - `option_hash` = 옵션 조합/추가구성/커스텀 입력까지 포함한 **표준화 문자열의 해시**
        
    - 같은 상품이라도 옵션이 다르면 **별도 row** 유지, 같으면 **수량 가산**
    
*  이미 운영 중인 `shop_cart` 테이블에 `option_hash` 컬럼을 새로 추가하려는 상황이라면,
	
	* 존 데이터 row에는 아직 옵션 값이 없으니 `NOT NULL` 제약을 바로 걸면 **ALTER TABLE 시점에 에러**가 나거나 기본값이 필요합니다.    
	
	- 따라서 보통은 **nullable 허용 후, 마이그레이션으로 값 채우고 → 그 다음 NOT NULL + UNIQUE 제약**을 거는 순서를 밟습니다.
    

---

## 안전한 적용 순서

1. **컬럼 추가 (NULL 허용)**
    
```sql
ALTER TABLE shop_cart ADD COLUMN option_hash VARCHAR(64) NULL AFTER product_id;
```


2. **기존 데이터 채우기**
    
    - 단순 상품: `option_hash = ''` 또는 `MD5('')`
        
    - 옵션상품: product_option 테이블 join 해서 조합값 해시 채우기
        
```sql
UPDATE shop_cart SET option_hash = '' WHERE option_hash IS NULL
```


3. **NOT NULL로 바꾸기**

```sql
ALTER TABLE shop_cart MODIFY option_hash VARCHAR(64) NOT NULL DEFAULT '';
```


4. **UNIQUE 제약 추가**
    
```sql
ALTER TABLE shop_cart ADD UNIQUE KEY uniq_user_product_option (user_id, product_id, option_hash);
```


---

## 정리

- 이미 존재하는 테이블에선 `NOT NULL`을 바로 추가하지 말고 → `NULL 허용 → 값 채움 → NOT NULL 전환` 순서가 안전합니다.
    
- UNIQUE 인덱스도 데이터 정합성 맞춘 뒤 추가해야 충돌 없이 걸립니다.