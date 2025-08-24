---
layout: default
title: "[admin] 반품상세"
parent: shop
grand_parent: Web
nav_order: 1
---

# 쇼핑몰 어드민 반품 상세 페이지 개발

Here are admin detail page developmnet tips.

### 반품 관리 목록 페이지라면 사실상 대부분의 관리자 페이지가 그렇듯이:
* 주문번호, 반품번호(return_id)
* 회원ID/이름
* 상품명 (또는 대표 상품명 + "외 n건")
* 반품 상태(status)
* 신청일(request_date)
* 환불 금액(refund_amount)
   
👉 이런 요약 정보만으로도 목록 조회는 충분합니다.

### 하지만 상세 조회가 필요한 이유는 다음과 같아요:
1. 고객이 입력한 반품 사유/추가 설명 (reason, desc)  
  → 목록에 전부 보여주면 너무 길어져서 UI가 복잡해짐.

2. 첨부 이미지(image_url)  
  → 파손·불량 증빙 사진 같은 건 상세 페이지에서 확인하는 게 일반적임.

3. 회수 송장(tracking_number), 반품 주소(return_address)  
  → 물류 담당자가 확인할 때 필요.

4. 관리자 메모(admin_memo)  
  → 처리 내역 기록. 목록에서 다 노출하면 지저분해짐.

5. 처리 타임라인 (request_date, processed_date, refund_date)  
  → 목록에는 보통 "신청일"만, 상세에서 타임라인을 한눈에 확인.

### ✅ 정리하면:
* 목록 페이지: 검색/필터 + 요약 정보 (반품번호, 주문번호, 회원, 상품명, 상태, 신청일, 환불금액).
* 상세 페이지: 고객 사유, 첨부 이미지, 송장, 주소, 환불 처리 내역, 관리자 메모.
