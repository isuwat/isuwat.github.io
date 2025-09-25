---
layout: default
title: 스토리 포인트
parent: Jekyll
nav_order: 2
---



좋아, **스토리에 포인트(Story Points)** 설정하는 법과 이번 스토리에 권장 포인트까지 한 번에 정리할게.

# 어떻게 설정하나요? (Jira 기준)

- **팀 관리(Team-managed) 프로젝트**
    
    1. 프로젝트 설정 → **Features** → **Estimation** 켜기(Story points 선택).
        
    2. 스토리 이슈 열기 → 우측 패널의 **Story points** 필드에 값 입력.
        
    3. 필드가 안 보이면: Project settings → **Issue types → Story → Fields**에서 Story points 노출.
        
- **회사 관리(Company-managed) 프로젝트**
    
    1. 보드 설정(Board settings) → **Estimation** → Estimation statistic 를 **Story points**로 설정.
        
    2. 스토리 화면에 **Story points**가 안 보이면: Project settings → **Screens / Fields**에서 해당 화면에 Story points 필드 추가.
        
    3. 백로그에서 여러 스토리 **일괄 편집**으로 Story points 입력도 가능.
        

> **서브태스크**에는 보통 스토리 포인트를 넣지 않습니다. (시간은 서브태스크의 **Original estimate / Work log**로 관리)

# 어떤 스케일을 쓰나요?

- 권장 스케일: **Fibonacci**(0.5, 1, 2, 3, 5, 8, 13 …)
    
- 캘리브레이션 예:
    
    - **1**: 아주 단순한 UI 수정/카피 변경
        
    - **3**: 한 화면 + 단순 API 연동
        
    - **5**: FE + API + DB/SP 수정, 테스트 포함
        
    - **8**: 다중 시스템/권한/동시성 이슈, 리스크 큼
        

# 이번 스토리(“사용자 배송완료→구매확정”) 권장 포인트

- 범위: FE(버튼/모달 안정화) + API 엔드포인트 + DB 프로시저 + 로깅 + QA
    
- **추천: 5 포인트**
    
    - 이유: 다층 변경(FE/BE/DB) + 모달 스크립트 정리 + 멱등/락 처리, 하지만 외부 연계는 없음.
        
    - 리스크(모달 충돌/전역 스크립트 영향)가 크다고 느껴지면 **8pt**도 가능.
        

# 운영 팁

- **스토리**에만 포인트 넣고, **태스크**에는 시간(Original/Remaining/Work log)으로 추적.
    
- 포인트는 **AC(수용기준) 정리 후** 추정(= DoR 충족 시점).
    
- 스프린트 속도(velocity)는 스토리 포인트로만 집계 → 계획/예측에 일관성 유지.
    

필요하면 포인트 캘리브레이션 카드(팀 기준점)도 만들어줄게—말만 해!