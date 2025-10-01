---
layout: default
title: payload
parent: Cheat Sheet
nav_order: 4
---

간단히 말해 **페이로드(payload)** 는 **요청/응답에서 실제로 실어 나르는 데이터 본문**을 의미해요.  
우리 문맥에선 보통 **AJAX로 서버에 보내는 JSON 본문**, 또는 서버가 돌려주는 **JSON 본문**을 가리킵니다.

### 요청 페이로드 (클라이언트 → 서버)

```js
const payload = {
  product_name: "테스트 상품",
  sale_price: 12900,
  is_best: 1,
  best_rank: 10
};

$.ajax({
  url: './exec_register_adm_shop_product.php',
  method: 'POST',
  contentType: 'application/json; charset=utf-8',
  data: JSON.stringify(payload)   // ← 이 JSON 문자열이 "요청 페이로드"
});

```

### 응답 페이로드 (서버 → 클라이언트)

```php
echo json_encode([
  'ok' => true,
  'code' => 0,
  'msg' => 'created',
  'product_id' => 'SKU-00123'
], JSON_UNESCAPED_UNICODE);  // ← 이 JSON 본문이 "응답 페이로드"

```

## 페이로드 vs. 다른 것들

- **헤더(headers)**: 메타 정보(예: Content-Type). 페이로드는 아님.    
- **쿼리스트링**: `?limit=20` 같은 URL 파라미터. 엄밀히는 페이로드가 아니라 **URL 데이터**.    
- **본문(body)**: 페이로드가 실리는 자리. (POST/PUT/PATCH 등에서 주로 사용)
    

## GET/POST 관점

- **GET**: 보통 페이로드 없이 URL에 파라미터를 붙임.    
- **POST/PUT**: **본문에 페이로드**를 담아 보냄(폼데이터, JSON 등).    

## 우리 프로젝트에서의 포인트

- **JSON 규격**: 숫자/문자/`null` 구분 엄격. (빈 문자열 `""`과 `null`은 다름)    
- **미전달 vs null**:    
    - _미전달(키 없음)_ → “변경 없음” 의미로 쓰기 좋음.        
    - `null` → “값을 비움/초기화” 로 약속하기 좋음.  
        프런트에서 `undefined` 속성은 `JSON.stringify` 시 자동으로 빠져서 **미전달**이 됩니다.
        
- **보안**: 비밀/토큰은 페이로드에 넣지 않기(특히 URL 쿼리스트링에도 X).    
- **검증**: 서버에서 타입·범위 체크(예: 정렬은 0 이상 정수).    
- **크기**: 너무 큰 페이로드(대용량 문자열/이미지)는 지양. 이미지/파일은 업로드로 처리.    

## 짧은 체크리스트

- 필요한 필드만 보낸다(최소화).    
- 타입 맞춘다(숫자/문자/불리언/`null`).    
- “미전달”과 `null` 의미를 서버와 합의.    
- 서버는 실패 시 **일관된 응답 페이로드**(예: `{ok:false, code, msg}`)를 준다.    

요 정도로 이해하면, “페이로드”라는 말이 나올 때 **본문에 담기는 실제 데이터**를 떠올리면 됩니다.