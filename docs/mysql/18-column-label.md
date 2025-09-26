---
layout: default
title: "[MySQL] 컬럼 코드와 라벨" 
parent: MySQL
nav_order: 18
---



`order_user_type_label`처럼 “라벨” 컬럼명을 원본 `order_user_type`와 다르게 두는 이유는:

- **값의 의미/타입이 다름**:
    
    - `order_user_type` = 숫자 코드(1,2…) — 필터, 정렬, 통계에 사용
        
    - `order_user_type_label` = 표시용 문자열(“자동”, “관리자”) — UI 렌더링용  
        같은 이름으로 덮어쓰면 숫자가 문자열로 바뀌어 정렬/집계가 꼬일 수 있어요.
        
- **중복 별칭 충돌 방지**:  
    `SELECT v.*, CASE … END AS order_user_type`처럼 같은 이름을 쓰면,  
    많은 드라이버/프레임워크(PHP의 `PDO::FETCH_ASSOC` 등)에서 **키 충돌**로 마지막 값이 전자를 덮어씁니다(= 숫자 코드가 사라짐).
    
- **안전한 API 스키마**:  
    클라이언트는 항상 `*_code`로 로직을, `*_label`로 표시를 쓰게 해두면 **역호환**이 쉽고 버그가 줄어요. (다국어 확장 시 `*_label_ko`, `*_label_en`도 깔끔)
    
- **정렬/인덱스 보존**:  
    정렬·필터는 `order_user_type`(숫자) 기준으로 하고, 화면에는 `order_user_type_label`을 쓰면 **인덱스 활용**과 **표시 품질**을 둘 다 챙깁니다.

### 추천 패턴

```sql
SELECT
  v.order_user_type              AS order_user_type,        -- 숫자 코드(그대로 유지)
  CASE v.order_user_type
    WHEN 1 THEN '자동'
    WHEN 2 THEN '관리자'
    ELSE 'N'
  END                           AS order_user_type_label,   -- 표시용 라벨
  v.price,
  FORMAT(v.price, 0, 'ko_KR')   AS price_fmt                -- 숫자/표시 분리(정렬은 v.price로)
...

```

요약: **코드와 라벨은 용도가 달라서 이름을 분리**하면 타입 혼선·키 충돌·정렬 문제를 예방하고, 프런트/백엔드 모두 깔끔해집니다