---
layout: default
title: 검색 쿼리
parent: Jekyll
nav_order: 4
---



gi
## 자주 쓰는 조합

- 현재 프로젝트의 미완료 스토리
```sql
project = ABC AND issuetype = Story AND statusCategory != Done ORDER BY updated DESC
```

* 현재 스프린트의 스토리
```sql
project = ABC AND issuetype = Story AND sprint in openSprints()

```

* 특정 에픽에 속한 스토리
```sql
project = ABC AND issuetype = Story AND "Epic Link" = ABC-123

```

* 내게 할당된 스토리
```sql
assignee = currentUser() AND issuetype = Story AND statusCategory != Done

```

## 어디서 설정?

- **고급 이슈 검색(Advanced search)** 화면 → **JQL**에 위 쿼리 입력  
    (기본 모드라면 상단 필터에서 **Project=ABC**, **Issue Type=Story** 선택)
    

## 보드에서 “빠른 필터”로 쓰고 싶다면

- 보드 설정 → **Quick filters** → 새 필터 추가
    
    - Name: `Stories`
        
    - JQL: `issuetype = Story`
        

## 참고

- 한국어 프로젝트에서 이슈 타입 이름이 “스토리”로 보일 수도 있어요. JQL에선 보이는 이름 그대로 쓰면 됩니다:  
    `issuetype in (Story, 스토리)`
    
- 여러 타입 묶어 보기:  
    `issuetype in (Story, Bug)`