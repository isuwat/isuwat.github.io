---
layout: default
title: "[MySQL] out parameter"
parent: MySQL
nav_order: 4 
---

# OUT 파라미터 

## OUT 파라미터 방식

보통은 이렇게 정의합니다:

```sql
CREATE PROCEDURE sample_proc(OUT po_rc INT, OUT po_msg VARCHAR(255)) 
BEGIN     
SET po_rc  = -1;     
SET po_msg = '에러 발생'; 
END;
```


호출:

`CALL sample_proc(@rc, @msg); SELECT @rc, @msg;`

👉 OUT 파라미터를 바인딩한 세션 변수에 결과가 자동으로 들어갑니다.

---

## 3. 차이점 정리

|방식|장점|단점|
|---|---|---|
|**OUT 파라미터**|공식적인 인터페이스 (명확, 유지보수 용이)|호출 시 반드시 세션 변수를 매핑해야 함|
|**세션 변수 직접 세팅**|OUT 파라미터 없이도 외부 조회 가능|인터페이스가 불명확, 다른 세션에 영향 없음 (세션 단위라서 안전)|

---

✅ **정리**

- OUT 파라미터를 쓰지 않고도 값을 확인하려면, 프로시저 안에서 `@rc`, `@msg` 같은 **세션 변수를 직접 SET** 해주면 됩니다.
    
- 하지만 권장 방식은 **OUT 파라미터 사용**이에요. (왜냐하면 프로시저 인터페이스가 명확해지고, 협업/유지보수가 쉬워지거든요.)


PHP(PDO)에서 **프로시저 OUT 값 확인**을 두 가지 방식으로 보여드릴게요.

---

## 1️⃣ OUT 파라미터 방식 (권장)

프로시저 정의:

```sql
CREATE PROCEDURE shop_cart_selected_order_form(
    IN  pi_user_id VARCHAR(20),
    IN  pi_cart_ids JSON,
    IN  pi_scope VARCHAR(20),
    OUT po_result_code INT,
    OUT po_err_msg VARCHAR(255)
)
BEGIN
    -- 로직 ...
    SET po_result_code = 0;
    SET po_err_msg     = 'All operations successful';
END;

```


PHP 호출:
```php
// 1) 프로시저 실행하면서 OUT 파라미터를 세션 변수에 매핑
$stmt = $dbh->prepare("CALL shop_cart_selected_order_form(:uid, CAST(:ids AS JSON), :scope, @rc, @msg)");
$stmt->execute([
    ':uid'   => $user_id,
    ':ids'   => $json_ids,
    ':scope' => $scope,
]);

// 2) OUT 값 조회
$res = $dbh->query("SELECT @rc AS rc, @msg AS msg")->fetch(PDO::FETCH_ASSOC);

echo $res['rc'];   // 0
echo $res['msg'];  // All operations successful

```


---

## 2️⃣ 세션 변수 직접 세팅 방식 (OUT 파라미터 생략 가능)

프로시저 내부:

```sql
-- OUT 파라미터가 있어도 되고 없어도 됨
SET @rc  = 0;
SET @msg = 'All operations successful';

```

PHP 호출:
```php
// 1) 프로시저 호출 (OUT 파라미터 매핑 안 해도 됨)
$stmt = $dbh->prepare("CALL shop_cart_selected_order_form(:uid, CAST(:ids AS JSON), :scope, @dummy1, @dummy2)");
$stmt->execute([
    ':uid'   => $user_id,
    ':ids'   => $json_ids,
    ':scope' => $scope,
]);

// 2) 내부에서 SET 한 세션 변수 조회
$res = $dbh->query("SELECT @rc AS rc, @msg AS msg")->fetch(PDO::FETCH_ASSOC);

echo $res['rc'];   // 0
echo $res['msg'];  // All operations successful

```


---

## ✅ 차이점

- **OUT 파라미터 방식**: 인터페이스가 명확 → 협업/유지보수에 유리.
    
- **세션 변수 직접 세팅**: 호출이 간단 → 개발/테스트 편의용으로 좋음.