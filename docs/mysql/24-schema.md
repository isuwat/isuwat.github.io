---
layout: default
title: "[MySQL] 스키마(schema)" 
parent: MySQL
nav_order: 24
---



DB에서 말하는 _스키마(schema)_ 는 “데이터베이스 구조(청사진)”이고, **DDL**은 그 구조를 만들고 바꾸는 **언어/명령**입니다. 스키마≠DDL, 관계는 “설계도 ↔ 설계도를 만드는/수정하는 도구”에 가깝습니다.

### 스키마가 담는 것 (무엇의 ‘모양’인가)

- 테이블과 컬럼(이름, 데이터 타입, NULL 여부, 기본값)
    
- 제약조건(PK/FK/UNIQUE/CHECK)
    
- 인덱스, 뷰, 시퀀스, 트리거/프로시저 등 오브젝트
    
- 객체 간 관계(참조 무결성)
    

### DDL은 무엇?

- **D**ata **D**efinition **L**anguage: 스키마를 정의/변경/삭제하는 SQL
    
    - `CREATE/ALTER/DROP TABLE`
        
    - `CREATE/DROP INDEX`
        
    - `CREATE SCHEMA`(또는 `CREATE DATABASE`), `CREATE VIEW` …
        

### 예시 (MySQL)

```sql
-- 스키마(=데이터베이스) 생성
CREATE SCHEMA shop;  -- MySQL에선 CREATE SCHEMA = CREATE DATABASE

-- 스키마의 한 조각(테이블) 정의
CREATE TABLE voucher (
  seq_no        INT PRIMARY KEY,
  regdate       DATETIME NOT NULL,
  user_id       VARCHAR(50),
  restore_user_id VARCHAR(50),
  price         BIGINT NOT NULL,
  stat          TINYINT NOT NULL,
  order_user_type TINYINT NOT NULL
);

-- 인덱스(스키마 요소)를 DDL로 생성
CREATE INDEX ix_voucher_regdate_seq ON voucher (regdate, seq_no);

```

위처럼 **DDL을 실행한 결과물이 스키마**입니다.

### DBMS마다 쓰는 말 차이 (요점만)

- **MySQL**: _schema ≒ database_ (거의 동의어)
    
- **PostgreSQL/SQL Server**: “데이터베이스” 안에 “스키마(네임스페이스)”가 있고, `public.sales`처럼 `schema.object`로 씀
    
- **Oracle**: 사용자(소유자)가 가진 객체 집합을 스키마라고 부름
    

### 정리

- “DB 스키마” = 구조 그 자체(ERD로도 문서화 가능)
    
- “DDL” = 그 구조를 만들고 고치는 SQL 명령