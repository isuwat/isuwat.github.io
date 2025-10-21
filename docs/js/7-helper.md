---
layout: default
title: "[JS] helper"
parent: JS
nav_order: 7 
---


### 별칭(helper) 함수


`const $ = (id)=>document.getElementById(id);` 는 **짧은 별칭(helper) 함수**예요.

### 한 줄 설명

- `$('someId')`라고 쓰면 `document.getElementById('someId')`를 호출한 것과 **같다**.
    

### 왜 쓰나?

- **타이핑 절약**: 긴 `document.getElementById` 대신 `$`로 간단히.
    
- **가독성**: 코드 전반에서 DOM 조회가 많을 때 읽기 쉬움.
    

### 어떻게 동작?

- 화살표 함수 `=>`로 함수를 정의.
    
- `id` 하나를 받아, 해당 id를 가진 **단일 요소**를 반환(없으면 `null`).

```js
const titleEl = $('page_title');     // == document.getElementById('page_title')
if (titleEl) titleEl.textContent = '주문서';

```
### 참고

- `getElementById`는 **앞에 `#` 없이** id 문자열만 전달해야 함.
    
- CSS 선택자를 쓰는 `querySelector('#id')`와 달리 이건 오직 **id 전용**이라 빠르고 명확.
    
- 전역 `$`를 다른 라이브러리(jQuery 등)도 쓸 수 있으므로, 충돌 우려가 있으면 이름을 바꿔도 됨:

```js
const byId = (id) => document.getElementById(id);

```

