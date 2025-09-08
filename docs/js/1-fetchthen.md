---
layout: default
title: "[JS] fetch().then()"
parent: JS
nav_order: 1
---


# fetch().then()

`fetch().then()`은 **JavaScript에서 비동기 HTTP 요청을 보내고 응답을 처리할 때 사용하는 방법**입니다. `fetch()`는 **Promise를 반환**하며, `.then()`을 사용해 순차적으로 결과를 처리할 수 있습니다.

✅ 기본 구조

```js
fetch('https://api.example.com/data')
  .then(response => {
    // 응답 객체를 처리
    return response.json(); // JSON 형태로 파싱
  })
  .then(data => {
    // 파싱된 데이터 사용
    console.log(data);
  })
  .catch(error => {
    // 오류 처리
    console.error('에러 발생:', error);
  });

```

## 🧩 각 단계 설명

### 1. `fetch(url)`

- 지정한 URL로 **HTTP 요청**을 보냄
    
- `Promise<Response>` 객체를 반환
    

### 2. `.then(response => {...})`

- 서버로부터 응답을 받으면 실행
    
- `response.ok`가 `false`인 경우 오류일 수 있으니 확인 필요
    
- 응답 본문을 `.json()`, `.text()`, `.blob()` 등으로 **파싱해서 다시 Promise로 반환**
    

### 3. `.then(data => {...})`

- 파싱된 실제 데이터를 받을 수 있는 단계
    
- API 결과물을 사용해 화면에 출력하거나 가공
    

### 4. `.catch(error => {...})`

- 네트워크 에러나 JSON 파싱 오류 등 발생 시 실행되는 **에러 처리 핸들러**

---

📝 예시: 에러 체크 추가

```js
fetch('https://api.example.com/user')
  .then(res => {
    if (!res.ok) {
      throw new Error('서버 응답 오류: ' + res.status);
    }
    return res.json();
  })
  .then(data => {
    console.log('유저 데이터:', data);
  })
  .catch(err => {
    console.error('요청 실패:', err);
  });

```

## 💡 요약

| 개념                | 설명                             |
| ----------------- | ------------------------------ |
| `fetch()`         | 비동기 HTTP 요청 시작                 |
| `.then()`         | Promise가 이행되었을 때 실행            |
| `.catch()`        | 요청 실패 또는 예외 발생 시 실행            |
| `response.json()` | 응답을 JSON으로 파싱 (또는 `.text()` 등) |


좋은 질문입니다. `fetch` 사용 시 **첫 번째 `.then()`에서 실제 데이터를 처리하지 않는 이유**는 다음과 같은 **비동기 처리 구조와 응답 객체(`Response`)의 특성 때문**입니다.

---

## 📌 요점 정리:

**`fetch()`의 첫 번째 `.then()`에서는 "응답(Response) 객체"를 받고, 그 안의 실제 데이터는 "두 번째 `.then()`에서" 처리합니다.**

---

## 🧠 왜 그런가요?

### 1. `fetch()`는 네트워크 요청을 보내고,

### 👉 첫 번째 `.then()`에서 `Response` 객체가 반환됩니다.

```js
fetch('https://api.example.com/data')
  .then(response => {
    console.log(response); // 아직 실제 JSON 데이터가 아님!
    return response.json(); // 본문(body)을 파싱하는 비동기 작업
  })

```

- 이 `response`는 HTTP 응답에 대한 전체 정보(헤더, 상태 코드 등)를 포함합니다.
    
- **본문(body)은 아직 문자열/스트림 형태로 남아 있어서 직접 접근 불가**합니다.


### 2. `response.json()`도 **비동기 작업이므로**, 또 `.then()`을 써야 합니다.

```js
  .then(data => {
    console.log(data); // 이제 실제 JSON 데이터!
  });

```

- `response.json()`은 **본문을 JSON으로 파싱하는 또 다른 Promise**를 반환합니다.
    
- 따라서 실제 데이터를 사용하려면 **한 단계 더 `.then()`**이 필요한 구조입니다.

### 🔄 요약 흐름도

```scss
fetch(url)
  ↓
.then(response => response.json())   ← 응답 자체 (아직 데이터 아님)
  ↓
.then(data => { ... })               ← 여기서 진짜 JSON 데이터 사용

```

## 🧪 비유로 이해하기

- `fetch()`는 택배를 시키는 것
    
- 첫 번째 `.then()`에서 택배 상자를 받음 (겉박스 = `Response`)
    
- `.json()`은 상자를 열어 내용물을 꺼내는 작업
    
- 두 번째 `.then()`에서 실제 상품(데이터)을 다루는 것

### 👀 참고로 async/await 버전은 이렇게 생겼어요:

```js
try {
  const response = await fetch(url);
  const data = await response.json();
  console.log(data);
} catch (err) {
  console.error('에러:', err);
}

```
