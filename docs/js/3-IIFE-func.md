---
layout: default
title: "[JS] func()"
parent: JS
nav_order: 3
---

# 즉시 실행 함수 표현식(IIFE, Immediately Invoked Function Expression)


```js
<script>
(function() {
  function getProductIdFromQS(){
    const p = new URLSearchParams(location.search);
    return p.get('product_id');
  }

  function addToCart(){
    const product_id = getProductIdFromQS();
    const qty = parseInt(document.getElementById('quantity').value || '1', 10);

    $.ajax({
      url: '../api/cart_proc.php',
      method: 'POST',
      dataType: 'json',
      data: { action: 'add', product_id, qty },
      success: function(res){
        if(res.ok){
          alert('장바구니에 담았습니다.');
        }else{
          alert('실패: ' + (res.err_msg || '알 수 없는 오류'));
        }
      }
    });
  }

  // 버튼 이벤트 연결
  $('#actionCart, #btnAddToCart').on('click', addToCart);
})();
</script>
```
### ✅ 구조 설명

1. (function() { ... })();

    function() { … } → 익명 함수 정의

    (...) → 괄호로 감싸서 식(Expression)으로 만듦

    (); → 바로 실행시킴

    결과: 함수가 정의되자마자 실행되고, 그 안의 로컬 스코프가 만들어짐

2. 내부에 정의된 함수 (getProductIdFromQS, addToCart)와 변수들은
    → 밖으로 노출되지 않고 이 블록 안에서만 살아있음
    → 전역 스코프(window)에 쓰레기 변수 안 남김 (오염 방지)

3. 내부 마지막 줄에서 이벤트 핸들러 연결:
```js
$('#actionCart, #btnAddToCart').on('click', addToCart);
```

→ DOM에 이미 있는 버튼에 클릭 이벤트를 걸어주는 부분.

### ✅ 왜 이렇게 쓰나?

- 이름 충돌 방지: 큰 프로젝트에서 전역에 함수를 막 흩뿌리면 충돌하기 쉽습니다.

- 모듈화 효과: 하나의 파일/기능 단위를 스코프 안에서 독립적으로 관리할 수 있습니다.

- 즉시 실행: DOM 로드 이후 바로 실행해야 하는 초기화 코드를 담기에 적합합니다.

### 📌 한 줄로 정리

이 형태는  

> “자기만의 스코프를 가진 작은 모듈을 선언하자마자 실행하는 패턴”

이라고 이해하시면 됩니다.


나의 말:
함수가 정의되자마자 실행됐는데, 이벤트 핸들러는 남아있나?
ChatGPT의 말:

아주 중요한 포인트 짚으셨어요 👏

### ✅ IIFE(즉시 실행 함수) 내부에서 이벤트 핸들러 등록 과정

예제:
```js
(function() {
  function addToCart(){
    alert("장바구니 담기");
  }

  // 실행 시점에, 버튼에 이벤트 핸들러를 "등록"
  $('#actionCart').on('click', addToCart);
})();
```

1. (function(){ ... })();
    → 이 블록이 한 번 실행됩니다.

2. 실행되면서 addToCart 함수가 지역 함수로 생성됨.

3. $('#actionCart').on('click', addToCart);
    → jQuery가 #actionCart 요소에 이벤트 핸들러를 등록합니다.

    이 때 등록된 건 addToCart 함수의 **참조(reference)**예요.

    이벤트 시스템 내부에 function 포인터가 저장됩니다.

4. IIFE 실행이 끝나면 블록 스코프 안의 addToCart 이름은 사라지지만,  
    이벤트 시스템이 addToCart 함수 객체를 참조하고 있기 때문에 GC(가비지 컬렉션)되지 않고 살아있습니다.

### ✅ 결론

- 함수 정의는 IIFE 안에서만 있었지만, 이벤트 핸들러로 등록되면서 DOM 요소와 함께 참조가 남음 
    → 클릭할 때마다 정상 실행됩니다.

- 그래서 "실행 직후 함수 스코프가 사라져도 이벤트 핸들러는 살아있나?" → 네, 살아있습니다.

### 🔑 기억하기

- 함수 객체(Function Object)와 함수 이름은 다릅니다.

    - 이름(addToCart)은 스코프가 끝나면 사라질 수 있음

    - 하지만 이벤트 시스템이 함수 객체 자체를 잡고 있기 때문에 메모리에 남아 있음

- 그래서 클로저(closure) 개념으로 이해하면 더 명확해요.