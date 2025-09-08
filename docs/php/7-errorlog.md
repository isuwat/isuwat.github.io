---
layout: default
title: PHP error_log()
parent: PHP
nav_order: 7
---



# PHP error_log()

```bash
grep /etc/php/7.x/apache2/php.ini
```


```bash
xxx@dev-server:/var/www/html/main$ grep error_log /etc/php/8.3/apache2/php.ini 
; server-specific log, STDERR, or a location specified by the error_log
;error_log = php_errors.log
;error_log = syslog
; to syslog. Only used when error_log is set to syslog.
; the message. Only used when error_log is set to syslog.
; OPcache error_log file name. Empty string assumes "stderr".
;opcache.error_log=
```

네, 보여주신 php.ini 일부를 보면:
```ini
;error_log = php_errors.log
;error_log = syslog

```


이렇게 **주석 처리**(`;`) 되어 있어서 현재는 `error_log` 경로가 지정되지 않은 상태예요.

### PHP 동작 규칙

- `error_log` 지시자가 주석 처리되면, PHP는 기본적으로 **웹서버의 에러 로그로 출력**합니다.
    
    - Apache mod_php → `/var/log/httpd/error_log` (CentOS/RHEL) 또는 `/var/log/apache2/error.log` (Debian/Ubuntu)
        
    - Nginx + PHP-FPM → PHP-FPM의 로그(`/var/log/php7.x-fpm.log` 등)나 Nginx의 `error.log`
        

즉, 지금 상태에서는 `error_log("메시지")`를 호출하면 **웹서버 에러 로그**에서 메시지를 확인할 수 있습니다.

---

### 확인 방법

1. `phpinfo();` 실행 → `error_log` 항목 확인 (지금은 아마 "no value" 나올 것)
    
2. 웹서버 로그 tail 해서 직접 확인:
```bash
tail -f /var/log/httpd/error_log         # CentOS/RedHat
tail -f /var/log/apache2/error.log       # Ubuntu/Debian
tail -f /var/log/nginx/error.log         # Nginx

```


---

### 원하는 로그 파일에 쓰게 하려면

php.ini에서 주석 해제하고 절대 경로로 지정:

```ini
error_log = /var/log/php_errors.log
```


그 후 웹서버/ FPM 재시작:

```bash
systemctl restart httpd      # Apache
systemctl restart nginx php-fpm

```
👉 지금 상태 그대로 쓰시면 **아파치/엔진엑스 에러로그 파일**에서만 확인 가능하고,  
명시적으로 파일을 따로 지정해야 `/var/log/php_errors.log` 같은 독립 파일로 쌓입니다.