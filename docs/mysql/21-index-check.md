---
layout: default
title: "[MySQL] INDEX 구성 검토" 
parent: MySQL
nav_order: 21
---


#### voucher INDEX 검증

```sql
CREATE INDEX ix_voucher_user_regdate_seq 
CREATE INDEX ix_voucher_restore_user_regdate_seq 
CREATE INDEX ix_voucher_regdate_seq ON voucher(regdate, seq_no);
```


**지금 만든 인덱스 3종**(regdate, user_id, restore_user_id 축) 기준으로,  
“키만 정렬 → 본문 JOIN(두 단계 페치)” 구조에 맞춘 **EXPLAIN 검증 스크립트**를 드릴게요.  
아래 쿼리들을 그대로 실행해서 `type / key / rows / Extra`를 확인하면 됩니다.

네, 그 3종 세트면 “현재 쿼리 패턴” 기준으로 깔끔한 베이스라인입니다.

- `ix_voucher_regdate_seq (regdate, seq_no)` → 전역 기간 조회 + 정렬/페이징(두 단계 페치, 키셋) 주력
    
- `ix_voucher_user_regdate_seq (user_id, regdate, seq_no)` → 사용자별 필터 + 정렬
    
- `ix_voucher_restore_user_regdate_seq (restore_user_id, regdate, seq_no)` → 복구자별 필터 + 정렬
    

이 구성으로 다음이 보장됩니다:

- 전역/기본 목록: `regdate, seq_no` 인덱스 range/역방향 스캔 → `filesort` 회피
    
- `user_id` 또는 `restore_user_id` 등식이 있을 때: 해당 복합 인덱스로 정렬까지 한 번에 처리
    

### 다만, 아래만 체크하세요

1. **중복 인덱스 정리**
    
    - 위 복합 인덱스가 있으면 `user_id` 단독, `regdate` 단독, `(user_id, regdate)` 같은 **선두-접두 인덱스는 대부분 불필요**합니다(Leftmost-prefix 규칙). 있으면 쓰기·메모리 낭비라 정리 추천.
        
2. **자주 쓰는 다른 필터가 있는지**
    
    - `stat=2` 같은 저선택도 코드 필터를 페이지마다 자주 건다면, 우선 서브쿼리에서  
        `USE INDEX (ix_voucher_regdate_seq)`로 플랜을 고정해 보세요.  
        그래도 느리면 **(stat, regdate, seq_no)** 보조 인덱스를 “정말 자주 쓸 때만” 추가.
        
3. **집계 전용 인덱스는 별도 판단**
    
    - `COUNT/SUM(price)`를 자주/넓은 구간에서 치면 `(regdate, price)` 커버링 인덱스가 효과 큼.  
        반대로 드물면 지금 구성으로도 충분; 인덱스는 최소 개수 유지가 원칙.
        
4. **EXPLAIN로 검증**
    
    - 전역 목록(필터 없음): `key=ix_voucher_regdate_seq`, `type=range`, `Using filesort` 없음
        
    - `user_id` 필터 목록: `key=ix_voucher_user_regdate_seq`
        
    - `restore_user_id` 필터 목록: `key=ix_voucher_restore_user_regdate_seq`
        
5. **쓰기 부하/버퍼 용량**
    
    - 90만 행 규모에서 복합 인덱스 3개는 보통 무난하지만, 버퍼풀 여유(<– 중요)와 쓰기 빈도 감안해 모니터링하세요.
        

### 결론

- 네, 현재 쿼리 패턴(전역 기간 조회 + 사용자/복구자 축 필터 + 정렬)에 딱 맞는 **균형 좋은 구성**입니다.
    
- 불필요한 단독/부분 인덱스가 있다면 정리하고, `stat`처럼 **특정 필터가 정말 잦을 때만** 추가 인덱스를 검토하면 됩니다.