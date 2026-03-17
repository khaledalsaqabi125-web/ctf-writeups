# Validator Write-up

## Challenge Info

- **Name:** Validator  
- **Category:** rev  
- **Difficulty:** Medium  

---

## Overview

The binary asks for a flag and validates it using an obfuscated check applied byte-by-byte.

Instead of comparing the full input directly, each character is validated independently using a hidden predicate function.

---

## Analysis

### Binary properties

- 64-bit ELF
- PIE enabled
- Not stripped → symbols are available

This makes reversing easier since function names are still present.

---

### Main logic

The validation flow is:

- `hIKCTDqsfNLU()` loops over all flag indices
- For each index, it calls:

```c
zWqDapvkXfHB(char byte, int idx)
```

This function determines whether a specific byte at a given position is correct.

---

### Internal mechanism

Inside the checker function:

- Several lookup tables are used from `.rodata`
- Total size ≈ 68 bytes (flag length)

Important tables:

- `XKCFcEiGsfwe`
- `LojuzgBPJdWU`

Additional tables used for obfuscation:

- `oKyKxnebFuod`
- `uJSFvJPHjoaB`

Each input byte is transformed using these tables and compared against expected values.

---

## Solving Strategy

### Key idea

Instead of guessing the entire flag, we invert the check:

- Treat the validation function as an **oracle**
- For each position:
  - Try all 256 byte values
  - Keep the one that passes the check

This reduces complexity drastically.

---

### Optimization

The function `wstLsACQERer()` introduces heavy obfuscation.

To speed up solving, we patch it in memory to return immediately:

```gdb
set {unsigned char}wstLsACQERer = 0xc3
```

This makes brute-forcing each byte instant.

---

## Solver (GDB Script)

```gdb
set pagination off
file /home/kali/validator
start

# patch heavy function
set {unsigned char}wstLsACQERer = 0xc3

python
import gdb
import re

length = 68
result = [0] * length
uncertain = {}

for i in range(length):
    valid = []
    for val in range(256):
        r = int(gdb.parse_and_eval(f"(int)zWqDapvkXfHB({val},{i})"))
        if r != 0:
            valid.append(val)

    if len(valid) == 1:
        result[i] = valid[0]
    else:
        uncertain[i] = valid

print("Uncertain positions:", len(uncertain))

charset = set(b"abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789{}_")

for i, vals in uncertain.items():
    printable = [v for v in vals if 32 <= v < 127]
    preferred = [v for v in printable if v in charset]

    if preferred:
        result[i] = preferred[0]
    elif printable:
        result[i] = printable[0]
    else:
        result[i] = vals[0]

flag_bytes = bytes(result)
text = flag_bytes.decode("latin1", errors="ignore")

print("Recovered:", text)

m = re.search(r"VBD\{.*?\}", text)
print("Flag:", m.group(0) if m else "Not found")
end

quit
```

---

## Result

The recovered flag is:

```text
VBD{I_kn0w_y0u_w0uld_us3_Opus_hehe_eafa09ad1898e0bcf9c0225076632225}
```

---

## Verification

```bash
echo 'VBD{I_kn0w_y0u_w0uld_us3_Opus_hehe_eafa09ad1898e0bcf9c0225076632225}' | ./validator
```

Output:

```text
Congratulations! You found the correct flag.
```

Thanks for reading and happy hacking!
