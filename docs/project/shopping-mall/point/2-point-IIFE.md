---
layout: default
title: "[point] 적립금 사용 - IIFE"
parent: point
nav_order: 1
---

# 적립금 사용 - IIFE



# 1) 요소 바인딩 (DOM 캐싱)

**무엇**  
화면에서 쓰는 핵심 노드를 한 번 잡아서 변수로 재사용.

```js
const $input  = document.getElementById('use_point'); // 적립금 입력
const $btnAll = document.getElementById('btnPointAll'); // 전액사용 버튼
const $used   = document.getElementById('ec-shop-payment_used_mileage_view'); // “-사용 적립금”
const $has    = document.getElementById('point_has'); // 보유 적립금(텍스트 또는 data-has)

// 총액 표시가 흩어져 있으니, 모두 갱신하기 위해 후보 ID 배열
const TOTAL_IDS = [
  'total_order_sale_price_view',
  'payment_total_order_sale_price_view',
  'pay_total_amount','final_amount','payment_total','order_total',
];

```

**왜**  
매번 `document.getElementById`로 찾지 않고 **1회만 참조**해서 성능/가독성 ↑.

**예제**  
버튼 안/본문의 총액이 여러 군데 있더라도 `TOTAL_IDS` 하나만 고치면 모두 동기화됨.

---

# 2) 헬퍼: 숫자/포맷/보유값/목록

**무엇**

```js
const onlyNumber = (s)=>{ s=String(s??'').replace(/[^\d]/g,''); return s?parseInt(s,10):0; };
const fmt = (n)=>(n||0).toLocaleString();
function getHas(){ return $has ? onlyNumber($has.getAttribute('data-has') || $has.textContent) : 0; }
function getTotalEls(){ return TOTAL_IDS.map(id=>document.getElementById(id)).filter(Boolean); }

```

**왜**

- **숫자 변환**: “12,300원” 같은 문자열을 숫자로 안전 변환
    
- **표시 포맷**: `10,000` 등 천단위
    
- **보유값**: `data-has` 우선, 없으면 텍스트
    
- **총액 노드 목록**: 한 번에 갱신하기 위한 리스트
    

**예제**  
`onlyNumber('12,300원') → 12300` / `fmt(12300) → '12,300'`

---

# 3) UX: 첫 포커스 시 0 지우기

**무엇**

```js
function clearIfZero(){ if ($input && $input.value.trim() === '0') $input.value = ''; }
$input?.addEventListener('pointerdown', clearIfZero, { passive: true });
$input?.addEventListener('focus', clearIfZero);

```

**왜**  
기본값 `0`이 있으면 500 입력 시 `0500` 같은 문제가 남. **첫 입력 편의성** 개선.

**예제**  
처음 터치/클릭하면 `0` → 빈칸으로 바뀌어 바로 숫자만 입력 가능.

---

# 4) 스냅샷: 기준 총액 고정(snapShotBases)

**무엇**

```js
function snapshotBases(){
  getTotalEls().forEach(el=>{
    if(!el.getAttribute('data-base')){
      el.setAttribute('data-base', String(onlyNumber(el.textContent)));
    }
  });
}

```

**왜**  
“중복 차감(누적 감소)” 방지. **항상 ‘최초 총액’(포인트 적용 전)**을 기준으로 계산해야 함.

**예제**

- 최초 총액 30,000 → `data-base=30000` 저장
    
- 이후 표시값이 25,000으로 바뀌어도, 다음 계산은 **항상 30,000 - 사용액**으로 수행
    

---

# 5) 기준 총액 읽기(getBase)

**무엇**

```js
function getBase(){
  const els = getTotalEls();
  for(const el of els){
    const v = onlyNumber(el.getAttribute('data-base') || el.textContent);
    if(v > 0) return v;
  }
  return 0;
}

```

**왜**  
총액 표시는 여러 군데가 있을 수 있으므로 **맨 처음 유효한** 기준을 사용.

**예제**  
버튼 안 엘리먼트가 먼저 존재하면 그 값을 기준으로 삼음.

---

# 6) 보정(clamp): 안전 범위로 자르기

**무엇**

```js
function clamp(v){
  const base = getBase();
  const has  = getHas();
  return Math.max(0, Math.min(v, base, has));
}

```

**왜**  
입력값이 과도해도 **음수 금지**·**보유/총액 상한 초과 금지**.

**예제**

- 보유 10,000 / 총액 25,000 / 입력 15,000 → 결과 **10,000**
    
- 보유 30,000 / 총액 25,000 / 입력 40,000 → 결과 **25,000**
    

---

# 7) 렌더(render): 한 번에 모두 갱신

**무엇**

```js
function render(){
  const base  = getBase();
  const used  = clamp(onlyNumber($input?.value));
  const final = Math.max(0, base - used);

  // 입력칸도 보정값으로 동기화
  if ($input && String(used) !== String($input.value)) $input.value = used;

  // (1) “-사용 적립금” 라인
  if ($used) $used.textContent = fmt(used);

  // (2) 총액(버튼/본문) 동시 갱신
  const finalText = fmt(final);
  getTotalEls().forEach(el => { if (el.textContent !== finalText) el.textContent = finalText; });

  // (3) 전액사용 버튼 상태: 전액이 이미 적용되었거나, 보유/총액이 0이면 비활성
  if ($btnAll) {
    const max = Math.min(getHas(), base);
    $btnAll.disabled = (max <= 0) || (used >= max);
    $btnAll.setAttribute('aria-disabled', String($btnAll.disabled)); // 접근성
  }
}

```

**왜**

- 어느 위치의 총액이든 **동일한 값**으로 즉시 반영
    
- 불필요한 더블 클릭 방지(전액 적용 상태면 버튼 비활성)
    

**예제**

- 기준 30,000 / 사용 5,000 → 최종 25,000
    
- 사용을 2,000으로 수정 → **다시** 30,000 - 2,000 = 28,000 (누적 감소 없음)
    

---

# 8) 입력 이벤트(onInput)

**무엇**

```js
function onInput(){ if(!$input) return; render(); }
$input?.addEventListener('input', onInput);

```

**왜**  
타이핑할 때마다 실시간으로 마이너스 라인/총액을 갱신.

**예제**  
500 → 1,000 → 15,000… 입력 변동마다 즉시 반영.

---

# 9) 전액사용 클릭 핸들러

**무엇**

```js
$btnAll?.addEventListener('click', () => {
  const max = Math.min(getHas(), getBase());
  if ($input) $input.value = clamp(max);
  render();
});

```

**왜**  
보유 vs 기준 총액 **중 작은 값**으로 곧바로 채워 UI 전체와 동기화.

**예제**  
보유 7,000 / 기준 12,000 → 전액사용 = **7,000** 입력, 총액은 12,000 - 7,000 = 5,000.

---

# 10) 초기화(최초 1회 실행)

**무엇**

```js
snapshotBases();
render();

```

**왜**  
페이지 로드 직후 **기준 고정** + **화면/상태 동기화**.

**예제**  
초기에 표시된 금액을 그대로 기준으로 삼아 이후 계산이 **항상 일정**해짐.

---

## 한 번에 보는 미니 예제 시나리오

- 초기 상태
    
    - 기준 총액(base) = **30,000** (스냅샷)
        
    - 보유(has) = **12,000**
        
- 사용자가 `use_point`에 `15000` 입력
    
    - `clamp(15000) = min(15000, 30000, 12000) = 12000`
        
    - 사용(used) = **12,000** / 최종(final) = **18,000**
        
    - 전액사용 버튼 = **비활성화**(이미 max 사용)
        
- 사용자가 `5000`으로 변경
    
    - `used=5,000` / `final=25,000`
        
    - 전액사용 버튼 = **활성화**(max 12,000에 미달)

## 실무 팁

- **총액 ID가 바뀌면** `TOTAL_IDS`만 수정하면 됨.
    
- **보유값 소스가 바뀌면** `#point_has`(또는 `data-has`)만 맞추면 됨.
    
- **딜레이**가 느껴지면 `#btnPointAll`/`.npay-input`에 `touch-action: manipulation;` 적용(이미 하셨음).