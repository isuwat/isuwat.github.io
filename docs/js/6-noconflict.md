---
layout: default
title: "[JS] noConflict(true)"
parent: JS
nav_order: 6
---



# 지금 상황에선 왜 문제였나

`common.js`가 `$`를 사용 중인데 `noConflict(true)`로 **$와 jQuery 둘 다 지워** 버려서,

> “$ is not a function”  
> 가 발생했어요. `Deferred/ready` 타이밍에도 `$`가 필요하거든요.

```html
	<script type="text/javascript" src="./js/jquery-3.6.0.min.js"></script>
	<script type="text/javascript" src="../js/common.js?ver=202412161430"></script>
	<!-- jquery 전역변수 선언 -->
	<script type="text/javascript">
		window.$JQ = jQuery.noConflict(true);
	</script>
```

# `noConflict()` vs `noConflict(true)`

- `noConflict()` (인자 없음): **전역 `$`만 반환/해제**, `window.jQuery`는 남김.
    
- `noConflict(true)`: **전역 `$`와 `window.jQuery` 둘 다 제거**하고, 그 jQuery 인스턴스(호출시점의 전역 jQuery)를 반환.
# 이 프로젝트에 권장 설정

- **가장 안전:** `noConflict(true)` 제거하고 전역 유지.

```html
<script src="./js/jquery-3.6.0.min.js"></script>
<script src="../js/common.js?ver=..."></script>
<script>
  // 필요하면 별칭만 제공
  window.$JQ = window.jQuery;
</script>

```

요약: `noConflict(true)`는 **특정한 충돌 상황**에서만 쓰는 스위치고, 지금처럼 `common.js`가 `$`를 기대하는 구조에선 쓰면 바로 깨집니다. 사용 목적이 “별칭 만들기”라면 전역 유지 + `window.$JQ = window.jQuery;`만으로 충분해요.


작은 예시 타임라인:

```html
<script src="/js/jquery-1.12.4.js"></script>
<!-- 지금 전역 jQuery는 1.12.4 -->

<script src="/js/jquery-3.6.0.js"></script>
<!-- 지금 전역 jQuery는 3.6.0 (1.12.4 → 3.6.0으로 덮임) -->

<script>
  var $jq3 = jQuery.noConflict(true);
  // 전역 복구:
  //   window.$, window.jQuery  ← 다시 1.12.4 (또는 기존 값/undefined)
  // 반환값:
  //   $jq3  ← 3.6.0 인스턴스 (여기에 .fn, .ajax 등 모두 붙어있음)
</script>

```

### 주의 (플러그인/공용 코드)

- **플러그인/공용 스크립트(common.js)**가 전역 `$`/`jQuery`를 기준으로 동작하면,  
    `noConflict(true)` 이후에는 **그 대상이 더 이상 3.6이 아니므로 깨질 수** 있어요.
    
- 이럴 땐:
    
    - 전역을 유지하고 `window.$JQ = window.jQuery;`만 두는 게 가장 안전