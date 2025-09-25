---
layout: default
title: 레이블
parent: Jira
nav_order: 1
---



짧게: **레이블은 “스토리에 기본”, 태스크/서브태스크엔 “필요 시 추가”**가 좋아요.

## 왜 이렇게?

- **보고/검색의 중심은 스토리**라서, 기능/도메인 기준 레이블(예: `myshop`, `orders`, `purchase-confirmation`)은 **스토리에 부여**하는 게 필터링·리포팅에 유리합니다.
    
- **태스크/서브태스크**는 구현 관점(예: `fe`, `api`, `db`, `sql`, `modal`, `alertjs`)처럼 **스토리와 다른 속성**이 있을 때만 추가하세요. 스토리와 동일하면 굳이 중복 넣을 필요 없습니다.
    

## 실무 추천 패턴

- **Epic**: 넓은 트랙이면 1~2개만(브랜드/제품 라인 수준). 너무 많이 붙이지 않기.
    
- **Story(기본)**: 기능/도메인 레이블 고정 세트(예: `myshop`, `orderlist`, `confirm-flow`).
    
- **Task/Sub-task(선택)**: 기술/작업유형 레이블만 추가(예: `fe`, `api`, `sp`, `sql`, `qa`).
    

## 자동화 팁(선택)

- **서브태스크 생성 시 스토리 레이블 자동 상속**(Jira Automation):
    
    - When: Issue created
        
    - If: `issueType = Sub-task`
        
    - Then: Edit issue → Labels = `{{parent.labels}}`
        
- 반대로 **태스크의 기술 레이블을 스토리로 올리고 싶지 않으면**(권장), 상속은 스토리→서브태스크만 걸고 역방향은 금지.
    

## 네이밍 & 운영 가이드

- **소문자-kebab-case**, 1이슈 **최대 3~5개** 권장.
    
- 고정 분류(팀/컴포넌트)는 **Labels 대신 Components** 필드 활용을 고려.
    
- 버전/릴리즈는 **Fix Version**으로, 기간은 **Sprint**로—레이블로 대체하지 않기.
    

### 이번 작업 예시

- Story: `myshop`, `orders`, `purchase-confirmation`
    
- Tasks:
    
    - FE: `fe`, `modal`, `alertjs`
        
    - API: `api`
        
    - DB/SP: `db`, `sp`, `sql`
        
    - QA: `qa`
        

요약: **스토리에 도메인 레이블 고정 → 태스크엔 기술 레이블만 필요 시 추가**가 가장 깔끔합니다.