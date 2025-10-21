---
layout: default
title: "[JS] 화살표 함수"
parent: JS
nav_order: 8
---


### 화살표 함수


**간단하게 함수 선언하는 최신 문법**이에요.  
`const $ = (id) => document.getElementById(id)` 처럼 **화살표 `=>`**를 써서 입력→출력 형태로 적습니다.

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

`(id) => document.getElementById(id)` 처럼 **화살표 `=>`**를 써서 입력→출력 형태로 적습니다.

### 기본 형태

```js
// 1개 인자 + 한 줄 반환(암시적 return)
const double = x => x * 2;

// 2개 인자
const add = (a, b) => a + b;

// 인자 없음
const now = () => Date.now();

// 여러 줄(명시적 return 필요)
const sum = (arr) => {
  let s = 0;
  for (const n of arr) s += n;
  return s;
};

```

# 왜 만들었나?

1. **보일러플레이트 줄이기**  
    기존:    
```js
const double = function (x) { return x * 2; };

```

	화살표:
```js
const double = x => x * 2;

```

- → “함수”를 엄청 자주 쓰는 JS에서 **타이핑/가독성**이 확 좋아집니다.
    
- **콜백/함수형 패턴에 맞춤**  
    `map, filter, reduce`처럼 함수를 인자로 넘기는 코드가 많아지면서, **짧고 한눈에 읽히는** 문법이 필요했어요.
```js
nums.filter(n => n > 0).map(n => n * 2);

```

**`this` 함정 피하기(렉시컬 this)**  
전통 함수는 호출 방식에 따라 `this`가 바뀌어요. 화살표 함수는 **자기만의 `this`를 만들지 않고**, 바깥(정의된 곳)의 `this`를 그대로 씁니다. 콜백 안에서 `this`가 튀는 문제를 줄여줘요.

```js
function Timer() {
  this.count = 0;
  setInterval(() => { this.count++; }, 1000); // 화살표: 바깥 this를 사용
}

```

# 언제 쓰면 좋은가?

- **짧은 콜백**: 이벤트, 배열 메서드(map/filter/reduce), Promise then/catch 등
    
- **내부에서 `this`를 안 바꾸고** 바깥 걸 그대로 쓰고 싶을 때
    
- **표현식 하나로 끝나는 로직** (암시적 return 덕분에 깔끔)

# 읽는 법(해석 요령)

- `(인자) => 표현식`  
    → “이 인자를 받아서 ‘표현식’의 값을 **그대로** 반환한다”
    
- `(인자) => { 여러 줄; return 값; }`  
    → 중괄호가 나오면 **명시적 return** 필요
    
- 인자 1개면 괄호 생략, 인자 0/2개 이상이면 괄호 필요


