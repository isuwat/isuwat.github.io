---
layout: default
title: "[main] best/md/new 섹션 데이터 바인딩"
parent: mypage
nav_order: 1
---


**Parent**: 홈 섹션 노출 v1 (BEST/MD/NEW; 기간 없음)  
**Story Points**: 2  
**Depends on**:
- S6: `select_shop_main_products` 프로시저 완료    
- v_products_effective 뷰 최신화(섹션 플래그/정렬 컬럼 포함)    
- DB 인덱스: `(is_best, best_rank, seq_no)`, `(is_md_pick, md_rank, seq_no)`, `(is_new, new_rank, seq_no)` (또는 product_id 정렬 사용 시 해당 키)    

---

## 목표(Goal)

메인 페이지 최초 로딩 시 서버사이드에서 `select_shop_main_products('ALL', N)`을 호출해 **BEST/MD/NEW** 섹션 상품을 받아 즉시 렌더링(SSR). 실패 시 사용자에게 팝업 안내.

---

## 범위(Scope)

1. **서버(PHP)**    
    - `main.php` 상단에서 SP 호출(SSR): 결과셋 3개 수집 → `$best/$md/$new`로 바인딩
    - 실패 시 `$home_error_msg` 세팅, 배열은 빈 값 처리        
    - (옵션) APCu 30~60초 캐시        
2. **템플릿**    
    - `<ul id="best-list">`, `<ul id="md-list">`, `<ul id="new-list">`에 PHP로 바로 출력        
    - `product_id`는 **VARCHAR**이므로 링크에 `rawurlencode()` 적용        
3. **에러 팝업**    
    - 본문 하단 스크립트에서 **ready/DOMContentLoaded** 시 `show_popup($home_error_msg)` 호출        
    - `show_popup()` 미정의 시 `alert()` 폴백        
4. **jQuery 충돌 방지**    
    - `noConflict(true)` **사용 금지** (전역 `$` 필요: `common.js`에서 `$` 사용)        
    - 별칭 필요 시 `window.$JQ = window.jQuery;`만 제공