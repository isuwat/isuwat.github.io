---
layout: default
title: PHP if 대체 제어구문
parent: PHP
nav_order: 3
---


# if 대체 제어구문

아, 네 🙂 그 부분은 **PHP의 대체 제어구문(alternative syntax)** 때문입니다.

---

### 🔹 기본 if 구문

보통은 중괄호 `{}` 로 블록을 열고 닫죠:

```php
if (!empty($row['seq_no'])) {     echo "상세보기"; }
```

---

### 🔹 대체 구문 (HTML 안에서 자주 씀)

PHP는 HTML과 섞어 쓰기 편하도록 `if: ... endif;` 같은 **콜론 문법**을 지원합니다:


```php
<?php if (!empty($row['seq_no'])): ?>     <a href="orderdetail.php?no=<?= (int)$row['seq_no'] ?>">상세보기</a> <?php endif; ?>
```

- `:` ← 블록 시작
    
- `endif;` ← 블록 끝
    

---

### 🔹 언제 쓰나?

- HTML이 섞여 있는 템플릿에서 **중괄호 대신** 쓰면 가독성이 좋아집니다.
    
- `if … endif;`, `foreach … endforeach;`, `for … endfor;` 등도 같은 방식으로 씁니다.
    

---

✅ 정리:  
`if (!empty($row['seq_no'])):` 의 `:` 는 **PHP 대체 구문에서 블록 시작을 알리는 기호**예요.  
닫을 때는 반드시 `endif;`를 써야 합니다.