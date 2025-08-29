---
layout: default
title: "[app] 자동 로그인 선택/해제"
parent: Web
nav_order: 4
---

# 자동 로그인 활성/비활성 로직 정리

## 1. 기본 개념

- **auto\_login\_flag** 값으로 자동 로그인 상태를 관리.

  - `0` → 자동 로그인 **활성화** 상태
  - `1` → 자동 로그인 **비활성화** 상태

- 사용자는 화면에서 라디오 버튼(활성/비활성) 중 하나를 선택 후 **확인 버튼** 클릭.

- 서버(DB)에는 회원별 `auto_login_flag` 값이 저장되어 있음.

---

## 2. 화면(UI) 처리

### 라디오 버튼 구성

```html
<label>
  <input type="radio" name="auto-login" value="0"> 활성화
</label>
<label>
  <input type="radio" name="auto-login" value="1"> 비활성화
</label>
```

- 페이지 로딩 시 DB에서 조회한 `auto_login_flag` 값에 따라 **라디오 버튼 자동 선택**.
  ```php
  <?php if ($auto_login_flag === '0') echo 'checked'; ?>
  ```

### 상태 메시지 표시

- `현재 자동 로그인 활성화 상태입니다.` (flag = 0)
- `현재 자동 로그인 비활성화 상태입니다.` (flag = 1)
- **활성화/비활성화 단어에 색상 부여**
  - 활성화 → 초록색
  - 비활성화 → 빨간색

---

## 3. 확인 버튼 클릭 로직 (프론트엔드)

### 함수 예시

```javascript
function onclick_register_member_auto_login_flag() {
  const selected = document.querySelector('input[name="auto-login"]:checked');
  if (!selected) {
    alert("자동 로그인 옵션을 선택하세요!");
    return;
  }

  const new_flag = selected.value;

  // 현재 DB에서 내려온 값과 비교 (PHP에서 주입)
  if (new_flag === current_auto_login_flag) {
    alert("현재 설정과 동일합니다. 변경할 필요가 없습니다.");
    return;
  }

  // 서버로 변경 요청
  fetch("update_auto_login_flag.php", {
    method: "POST",
    headers: {"Content-Type": "application/x-www-form-urlencoded"},
    body: "auto_login_flag=" + encodeURIComponent(new_flag)
  })
  .then(res => res.json())
  .then(data => {
    if (data.result_code === 0) {
      alert("자동 로그인 설정이 변경되었습니다.");
      current_auto_login_flag = new_flag; // 클라이언트 상태 갱신
    } else {
      alert("오류: " + data.err_msg);
    }
  })
  .catch(err => {
    console.error("통신 오류:", err);
    alert("요청 처리 중 오류가 발생했습니다.");
  });
}
```

---

## 4. 서버 로직 (PHP)

1. **POST 값 검증**

   ```php
   if (!isset($_POST['auto_login_flag'])) {
       return error(-4, "값이 전달되지 않았습니다.");
   }

   $new_flag = $_POST['auto_login_flag'];
   if ($new_flag !== '0' && $new_flag !== '1') {
       return error(-4, "잘못된 값이 입력되었습니다.");
   }
   ```

2. **DB 조회** (현재 flag)

   ```php
   $current_flag = get_auto_login_flag_from_db($user_id);
   ```

3. **비교 후 처리**

   - 동일하면 → 변경하지 않고 메시지 반환

   ```php
   if ($new_flag === $current_flag) {
       return success(1, "현재 설정과 동일합니다.");
   }
   ```

   - 다르면 → DB 업데이트

   ```php
   update_auto_login_flag($user_id, $new_flag);
   return success(0, "자동 로그인 설정이 변경되었습니다.");
   ```

---

## 5. 예외 처리 & 고려사항

- **empty() 사용 금지**: `0`도 empty로 처리되므로 `isset()`과 값 비교를 이용.
- **중복 요청 방지**: 같은 값으로 여러 번 요청하는 경우 UPDATE 생략.
- **보안**: 세션 사용자 ID와 POST로 전달된 값을 매칭해 검증 필요.
- **로그 기록**: 변경 내역을 DB 로그 테이블에 기록하면 추적 가능.

---

## 6. 전체 흐름 요약

1. 페이지 로딩 → DB에서 `auto_login_flag` 조회 → 라디오 버튼 선택 & 상태 메시지 출력
2. 사용자 선택 후 **확인 버튼 클릭**
3. JS에서 현재 값과 비교 → 다를 경우만 서버 요청
4. 서버에서 값 검증 + DB 업데이트
5. 성공 시 메시지 반환 & 클라이언트 상태 갱신

