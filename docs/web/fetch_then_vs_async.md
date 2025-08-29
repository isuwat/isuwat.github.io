---
layout: default
title: "[script] 비동기 요청"
parent: Web
nav_order: 5
---

# fetch + then vs fetch + async/await

JavaScript에서 비동기 요청을 처리하는 대표적인 두 가지 방식:
- `fetch + then`
- `fetch + async/await`

결과적으로 같은 일을 하지만, 문법과 가독성, 예외 처리 방식 등에서 차이가 있습니다.

---

## 1. 기본 개념

| 항목 | fetch + then | fetch + async/await |
|------|----------------|-----------------------|
| 방식 | Promise 체이닝 | 동기 코드처럼 표현 |
| 비동기 처리 | `.then()`으로 연결 | `await`로 기다림 |
| 에러 처리 | `.catch()` 사용 | `try/catch` 블록 |

---

## 2. 코드 비교

### ☕ fetch + then
```js
fetch('https://api.example.com/data')
  .then(response => response.json())
  .then(data => {
    console.log('Data:', data);
  })
  .catch(error => {
    console.error('Error:', error);
  });
```

### 🍱 fetch + async/await
```js
async function getData() {
  try {
    const response = await fetch('https://api.example.com/data');
    const data = await response.json();
    console.log('Data:', data);
  } catch (error) {
    console.error('Error:', error);
  }
}

getData();
```

---

## 3. 차이점 요약

| 항목 | fetch + then | fetch + async/await |
|------|--------------|---------------------|
| 가독성 | 중첩될 수 있음 | 깔끔하고 직관적 |
| 예외 처리 | 각 단계마다 분기 필요 | try/catch로 일괄 처리 |
| 코드 흐름 | 체이닝 방식 | 위→아래 순차 실행 |
| 반복문과 함께 사용 | 불편 | 매우 편리 |

---

## 4. 결론

- **같은 일을 다른 방식으로 처리하는 것**일 뿐이며,
- 최신 JavaScript에서는 **`async/await` 방식이 더 선호**됨.
- 간단한 작업에서는 `then`도 충분히 사용 가능.

---

## 💡 팁
- 비동기 로직이 많아질수록 `async/await` 방식이 코드 관리에 유리합니다.
- `fetch` 외 다른 비동기 API (예: axios 등)에도 동일하게 적용됩니다.

---

## 🚫 동기 방식은 왜 잘 안 쓸까?

- **동기(Synchronous)** 코드는 작업이 끝날 때까지 **전체 흐름이 멈추기 때문에**,
  - UI가 멈추거나,
  - 전체 앱이 느려질 수 있음.
- 특히 **브라우저 환경에서는 동기 코드가 사용자 경험을 해칠 수 있음**.
- 그래서 대부분의 네트워크 작업이나 I/O 작업은 **비동기(Promise 기반)** 로 처리하는 것이 표준입니다.

### ✅ 요약:
> JavaScript에서는 굳이 동기 코드를 쓸 필요가 거의 없습니다.
> 대신 `async/await`으로 동기처럼 보이게 작성하면 됩니다.

