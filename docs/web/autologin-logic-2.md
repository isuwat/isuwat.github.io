---
layout: default
title: "[app] 자동 로그인 선택/해제 상세"
parent: Web
nav_order: 5
---

# 자동 로그인 활성/비활성 로직 상세 정리

본 문서는 "자동 로그인" 기능의 **활성/비활성(켜기/끄기)** 로직을 프런트엔드(JS)와 서버(PHP) 양쪽 관점에서 **값 정의–초기화–검증–업데이트–표시**까지 일관되게 정리합니다.

---

## 0) 용어 & 값 정의

- **설정 값(Key)**: `auto_login_flag`
- **의미 매핑**
  - `0` → **활성화(켜기)**
  - `1` → **비활성화(끄기)**
- **라디오 버튼 값**: 끄기=`1`, 켜기=`0`
- **버튼**: 확인(`onclick_register_member_auto_login_flag()`), 취소(`onclick_cancel()`)

---

## 1) UI 구성 (라디오 + 상태 메시지 + 버튼)

- 라디오는 **같은 name**으로 단일 선택(`name="auto-login"`).
- 디자인: 원형 인디케이터 + 체크(✓) 표현. 포커스 outline 커스텀.
- 상태 문구는 **활성화/비활성화 단어만 색상** 처리.

### HTML 스니펫

```html
<div class="agree-radio">
  <label for="auto-off">
    <input type="radio" name="auto-login" id="auto-off" value="1" class="sr-only">
    <span class="indicator"></span>
    <span>끄기</span>
  </label>
  <label for="auto-on">
    <input type="radio" name="auto-login" id="auto-on" value="0" class="sr-only">
    <span class="indicator"></span>
    <span>켜기</span>
  </label>
</div>

<p class="status-msg" id="status-msg"></p>

<div class="btn-box">
  <button type="button" class="btn-confirm" onclick="onclick_register_member_auto_login_flag()">확인</button>
  <button type="button" class="btn-cancel" onclick="onclick_cancel()">취소</button>
</div>
```

### CSS 포인트

```css
.agree-radio { display:flex; gap:16px; margin:10px 0 12px; }
.agree-radio .sr-only{ position:absolute;width:1px;height:1px;margin:-1px;padding:0;border:0;clip:rect(0 0 0 0);clip-path:inset(50%);overflow:hidden;white-space:nowrap; }
.agree-radio label{ display:inline-flex; align-items:center; gap:8px; font-size:15px; color:#333; cursor:pointer; }
.agree-radio .indicator{ width:20px; height:20px; border-radius:50%; background:#f0f0f0; border:1px solid #ccc; position:relative; transition:background-color .3s,border-color .3s; }
.agree-radio input[type=radio]:checked + .indicator{ background:#4967a1; border-color:#4967a1; }
.agree-radio input[type=radio]:checked + .indicator::after{ content:""; position:absolute; top:2px; left:6px; width:4px; height:8px; border:solid #fff; border-width:0 2px 2px 0; transform:rotate(45deg); }
.agree-radio input[type=radio].sr-only:focus + .indicator{ outline:none; box-shadow:0 0 0 3px #cbd5e1; }

.status-msg { margin:6px 0 0; font-weight:bold; }
.status-msg .on  { color:green; }
.status-msg .off { color:red; }

.btn-box { display:flex; gap:10px; margin-top:12px; }
.btn-box button { padding:6px 14px; font-size:14px; border:1px solid #ccc; border-radius:4px; cursor:pointer; }
.btn-confirm { background:#4967a1; color:#fff; border-color:#4967a1; }
.btn-cancel { background:#f5f5f5; color:#333; }
```

---

## 2) 페이지 로드시 초기화 (PHP → JS/HTML)

> 서버에서 현재 저장된 설정값을 조회하여 라디오 상태와 상태 문구에 반영합니다.

### PHP 예시 (조회 + 라디오 checked + 상태 문구 생성)

```php
<?php
$return_params = exec_select_member_auto_login_flag($params);
$auto_login_flag = null;    // '0' 또는 '1'
$status_html = '';

if ($return_params['result_code'] === '0') {
  $auto_login_flag = (string)$return_params['auto_login_flag'];
}
?>

<!-- 라디오: PHP에서 직접 checked 반영 -->
<div class="agree-radio">
  <label for="auto-off">
    <input type="radio" name="auto-login" id="auto-off" value="1" class="sr-only" <?php if($auto_login_flag==='1') echo 'checked'; ?>>
    <span class="indicator"></span>
    <span>끄기</span>
  </label>
  <label for="auto-on">
    <input type="radio" name="auto-login" id="auto-on" value="0" class="sr-only" <?php if($auto_login_flag==='0') echo 'checked'; ?>>
    <span class="indicator"></span>
    <span>켜기</span>
  </label>
</div>

<p class="status-msg" id="status-msg">
  <?php if($auto_login_flag==='1'): ?>
    현재 자동 로그인 <span class="off">비활성화</span> 상태입니다.
  <?php elseif($auto_login_flag==='0'): ?>
    현재 자동 로그인 <span class="on">활성화</span> 상태입니다.
  <?php endif; ?>
</p>

<script>
  // JS에서도 현재값을 보관하여 "동일값 방지"에 사용
  var current_auto_login_flag = "<?= $auto_login_flag ?>"; // '0' or '1'
</script>
```

> **권장**: 가능하면 HTML 출력 시 `checked`를 직접 넣어 깜빡임 없이 초기화. JS로 넣을 경우 `DOMContentLoaded` 이후 체크되며 잠깐 미선택 상태가 보일 수 있음.

---

## 3) 확인 버튼 클릭 로직(JS) — 동일값이면 호출 차단

### JS 함수

```html
<script>
function setStatusMessage(val){
  const el = document.getElementById('status-msg');
  if (!el) return;
  if (val === '1') {
    el.innerHTML = '현재 자동 로그인 <span class="off">비활성화</span> 상태입니다.';
  } else if (val === '0') {
    el.innerHTML = '현재 자동 로그인 <span class="on">활성화</span> 상태입니다.';
  }
}

function onclick_register_member_auto_login_flag(){
  const selected = document.querySelector('input[name="auto-login"]:checked');
  if (!selected) { alert('자동 로그인 옵션을 선택하세요!'); return; }

  const new_flag = selected.value; // '0' 또는 '1'

  // 1) 동일값이면 서버 호출 차단
  if (new_flag === current_auto_login_flag) {
    alert('현재 설정과 동일합니다. 변경할 값이 없습니다.');
    return;
  }

  // 2) 서버로 업데이트 요청 (예: fetch)
  fetch('update_auto_login_flag.php', {
    method: 'POST',
    headers: { 'Content-Type': 'application/x-www-form-urlencoded' },
    body: 'auto_login_flag=' + encodeURIComponent(new_flag)
  })
  .then(r => r.json())
  .then(data => {
    if (data.result_code === 0) {
      alert('자동 로그인 설정이 변경되었습니다.');
      current_auto_login_flag = new_flag;  // 클라이언트 상태 갱신
      setStatusMessage(new_flag);          // 화면 문구 갱신
    } else if (data.result_code === 1) {
      // 서버에서도 동일값 감지 시(선택), 메시지 처리
      alert(data.err_msg || '변경 사항이 없습니다.');
    } else {
      alert('오류: ' + (data.err_msg || '처리 중 오류가 발생했습니다.'));
    }
  })
  .catch(err => {
    console.error(err);
    alert('통신 오류가 발생했습니다. 잠시 후 다시 시도해주세요.');
  });
}

function onclick_cancel(){
  // 예: 라디오 선택 해제(필요 시)
  document.querySelectorAll('input[name="auto-login"]').forEach(r => r.checked = false);
  // 또는 모달 닫기/뒤로가기 등
}
</script>
```

---

## 4) 서버: POST 값 검증 & 동일값 스킵 & 업데이트

### (1) 입력 존재 확인 + 화이트리스트 검증

```php
if (!isset($_POST['auto_login_flag'])) {
  $return_params = [
    'result_code' => -4,
    'err_msg' => '값이 전달되지 않았습니다.'
  ];
  echo json_encode($return_params); return;
}

$auto_login_flag_new = $_POST['auto_login_flag'];
if ($auto_login_flag_new !== '0' && $auto_login_flag_new !== '1') {
  echo json_encode(['result_code' => -4, 'err_msg' => '잘못된 값이 입력되었습니다.']);
  return;
}
```

> 주의: `empty()`는 `'0'`도 empty로 판단하므로 사용 금지. 항상 `isset()` + 허용값 검사로 처리.

### (2) 현재값 조회 → 동일하면 스킵

```php
$cur = exec_select_member_auto_login_flag($params);
if ($cur['result_code'] === '0') {
  $current_flag = (string)$cur['auto_login_flag'];
  if ($current_flag === $auto_login_flag_new) {
    echo json_encode([
      'result_code' => 1,
      'err_msg' => '현재 설정과 동일하여 변경이 필요 없습니다.'
    ]);
    return;
  }
}
```

### (3) 업데이트 실행

```php
$upd = exec_update_member_auto_login_flag($params, $auto_login_flag_new);
if (!empty($upd['success'])) {
  echo json_encode(['result_code' => 0, 'msg' => '자동 로그인 설정이 변경되었습니다.']);
} else {
  echo json_encode(['result_code' => -5, 'err_msg' => '업데이트 중 오류가 발생했습니다.']);
}
```

---

## 5) 상태 메시지 출력 규칙 (색상 분기)

- 문구 예: `현재 자동 로그인 <span class="on">활성화</span> 상태입니다.`
- **on**(활성화) → 초록, **off**(비활성화) → 빨강
- 전체 문장이 아닌 **핵심 단어만 색상** 적용

PHP 서버렌더 예시:

```php
if ($auto_login_flag === '1') {
  echo '<p class="status-msg">현재 자동 로그인 <span class="off">비활성화</span> 상태입니다.</p>';
} else if ($auto_login_flag === '0') {
  echo '<p class="status-msg">현재 자동 로그인 <span class="on">활성화</span> 상태입니다.</p>';
}
```

---

## 6) 엣지 케이스 & 예외 처리

- 서버 조회 실패: 상태 문구 대신 "설정을 불러올 수 없습니다" 표시. 확인 클릭 시에도 서버 오류 처리.
- 로그인 만료/권한 없음: 401/403 수신 시 로그인 페이지로 유도.
- 네트워크 타임아웃: 재시도 가이드(스낵바/알림) 제공.
- 다중 클릭 방지: 확인 버튼 클릭 시 로딩 상태/비활성 처리(선택).

---

## 7) 테스트 체크리스트

1. 페이지 진입 시 서버값이 라디오/문구에 정확히 반영되는가?
2. 확인 클릭 시 **동일값이면 서버 요청이 차단**되는가?
3. 값이 변경되면 서버 업데이트 및 **클라이언트 상태 재동기화** 되는가?
4. 잘못된 값(예: '2','true','') 전송 시 서버가 **-4**로 거절하는가?
5. 네트워크/서버 오류 시 사용자에게 적절히 안내되는가?
6. 키보드 탭 이동 및 스크린리더에서 라디오/버튼/상태 문구가 인식되는가?

---

## 8) 보안 & 안정성 메모

- 서버에서 **반드시** 값 검증(화이트리스트) 후 처리.
- XSS 방지: 직접 출력되는 텍스트는 escape. 상태 문구는 신뢰된 값으로만 innerHTML 작성.
- 로깅: 설정 변경 성공/실패 이벤트를 서버 로그에 남겨 추적 가능하도록.

---

## 9) 요약 플로우

1. 페이지 로드 → 서버에서 `auto_login_flag` 조회 → 라디오 checked/문구 표시
2. 사용자가 라디오 변경 → 확인 클릭
3. JS: `new_flag === current_auto_login_flag` 이면 **차단**
4. 다르면 POST 전송 → 서버: 검증 → 동일값 스킵 or 업데이트 → 결과 반환
5. 성공 시 JS에서 현재값/문구 갱신

---

필요 시 이 문서를 기준으로 **실행 파일(예: PHP include + JS 모듈)** 형태로 분리·정리해 드릴 수 있습니다.

