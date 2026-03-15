# PyFSB Write-up

**Challenge:** PyFSB  
**Category:** Pwn  
**Points:** 159  
**Solves:** 35  
**Author:** iwancof  

**Description**

Very Simple FSB

**Connection**

```
nc 35.194.108.145 13840
```

---

## 1. Bug

The bug is in `fsb.c`. The service reads one line from stdin and passes that line directly to `Py_BuildValue(request)`.

That means our input is not treated as plain data. Instead, it is interpreted as a Python C-API format string. As a result, CPython starts consuming fake varargs from registers and the stack.

This effectively gives us a format string style primitive through the Python C-API.

---

## 2. Exploit idea

First, I used a leak payload consisting of many `K` format units.  
On the remote service, `("K" * 28)` was enough to expose a stack pointer in the returned tuple.

Then I inspected the leaked values and found that one of them pointed close to the request buffer.

Using a small probe, I determined that:

```
buf = leak[9] - 0x100
```

---

## 3. Function pointer primitive

The second stage used the `O&` format unit.

In CPython, the `O&` specifier calls a function pointer with one attacker-controlled argument.

Because Ubuntu 24.04 `python3.12` is compiled **without PIE**, the address of `PyRun_SimpleString` remains constant:

```
0x4b5892
```

This allowed a clean call primitive:

```
PyRun_SimpleString("import glob; print(open(flag).read())")
```

---

## 4. Final payload

Stage 2 payload layout:

```python
fmt = b"KKKKKKKO&\x00" + b"A" * 6
payload = fmt + p64(0x4b5892) + p64(buf + 32) + python_code + b"\n"
```

The injected Python code searched for `flag-*` and printed its contents.

The service crashed with a `SystemError` afterward, which is expected because `PyRun_SimpleString` is not actually a `PyObject *` converter callback, but the side effect (executing the Python code) already happened.

---

## Flag

```
tkbctf{n3w_463_0f_f5b-805a5dd8f03016053bf77528ec56265b7c593e6612d54a458258e5e2eba51ab0}

Thanks for reading and happy hacking!
