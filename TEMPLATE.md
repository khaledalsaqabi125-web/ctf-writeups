# <Challenge Name> Write-up

## Challenge Info

- **CTF:** <competition name>
- **Category:** <Web | Pwn | Rev | Crypto | Forensics | Misc>
- **Difficulty:** <Easy | Medium | Hard>
- **Points:** <number, or omit>
- **Author:** <challenge author, or omit>
- **Target:** `<host:port or URL, or omit>`

---

## Description

> Paste the challenge description here, verbatim, as a blockquote.

---

## Summary

Two or three sentences: what the bug was, and how it led to the flag. A reader
should be able to stop here and still understand the solve.

---

## Step 1: Recon

What you looked at first, and what it told you. Keep commands in fenced blocks
and always tag the language:

```bash
curl -s http://target/status.php
```

Show the response when it matters:

```json
{"version":"10.13.0"}
```

## Step 2: <Finding the bug>

Explain the reasoning, not just the commands. What made you suspect this?

## Step 3: <Exploiting it>

```python
#!/usr/bin/env python3
import requests

# keep exploit code runnable — someone should be able to copy it and reproduce
```

---

## Root Cause

Why the bug existed in the code. One paragraph. This is the part that turns a
solve log into a write-up worth reading.

---

## Minimal Solve

The shortest path from nothing to flag, for someone who just wants to reproduce:

```bash
# 1. ...
# 2. ...
```

---

## Flag

```
FLAG{...}
```

---

## Key Takeaways

- What technique this challenge taught
- What you would look for next time to spot this class of bug faster

## Tools

`ffuf` · `Burp Suite` · `pwntools` — whatever you actually used.

---

<!--
  ═══════════════════════════════════════════════════════════════
  قواعد ثابتة لكل رايت-أب — احذف هذا التعليق من الملف النهائي
  ═══════════════════════════════════════════════════════════════

  ١. لازم يبدأ بـ H1 (سطر يبدأ بـ #) وكل قسم H2 (##).
     بدونها الصفحة تطلع كتلة نص وحدة بدون فهرس جانبي في GitHub.

  ٢. الصور: تنحط داخل مجلد assets/ جنب الرايت-أب، والمسار نسبي:
         ![وصف الصورة](assets/01_homepage.png)
     لا تستخدم أبداً مسار من جهازك مثل /home/user/... — يطلع مكسور للكل.

     الشكل النهائي:
         Exploit3rs/
         └── Black Site/
             ├── README.md
             └── assets/
                 ├── 01_homepage.gif
                 └── 02_login.gif

  ٣. أسماء المجلدات: بدون مسافة في آخر الاسم، وبدون امتداد وهمي
     مثل ".pdf" على مجلد. يفضّل حروف صغيرة وشرطات: black-site

  ٤. كل بلوك كود له لغة: ```bash ```python ```json ```http
     عشان تطلع الألوان صح.

  ٥. الفلاق يكون داخل بلوك كود، مو نص عادي — عشان ما ينكسر
     بسبب الشرطات السفلية والرموز.
-->
