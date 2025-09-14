---
layout: default
title: "[JS] 프런트값 정수 바인딩"
parent: JS
nav_order: 4
---

# 프런트 값 정수 바인딩



# 프런트 값 정수 바인딩

* incorrect integer value 오류

MySQL이 **빈 문자열(또는 숫자 아님)을 INT 컬럼에 넣으려 할 때** 특히 `STRICT_TRANS_TABLES` 모드에서 터지는 에러예요.  
입력값이 비었거나 공백/콤마가 섞인 경우를 **0으로 정규화**하고, **정수로 바인딩**하면 해결됩니다.

문자열 그대로 POST됨
```js
	sale_price: $('#sale_price').val().trim(),
	buy_price: $('#buy_price').val().trim(),
```
### 해결 방법

#### ① 프론트에서 0으로 변환

```js
const data = {
		sale_price: $('#sale_price').val().trim() || 0,
		buy_price: $('#buy_price').val().trim() || 0,
```

