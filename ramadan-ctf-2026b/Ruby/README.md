# Ruby Write-up

## Challenge Info

- **CTF:** VulnByDefault CTF  
- **Challenge:** Ruby  
- **Category:** Misc / Reverse Engineering  
- **Difficulty:** Medium  
- **Flag format:** VBD{...}  

---

## Step 1: Identify the Environment

Extracting the zip reveals a Minecraft world named:

```text
Computer v2 by mattbatwings
```

This is a fully functional **8-bit CPU** built using Minecraft redstone, known as **BatPU-2**.

The file `challenge.schem` is a WorldEdit schematic containing a program loaded into the CPU's ROM.

The gzip header reveals the original filename:

```text
solutionprogram.schem
```

---

## Step 2: Understand the Architecture

From the in-game signs and documentation:

- **16 opcodes:**

```text
NOP, HLT, ADD, SUB, NOR, AND, XOR, RSH,
LDI, ADI, JMP, BRH, CAL, RET, LOD, STR
```

- **16 registers:** `r0–r15`  
  (`r0` is always 0)

- **I/O ports:** memory addresses `240–255`

- **Instruction format:**  
  16-bit → `4-bit opcode + operands`

The official BatPU-2 repository includes:

- `assembler.py`
- `schematic.py`

These describe how instructions are encoded.

---

## Step 3: Decode the Program ROM

Using `schematic.py`, the encoding was reversed.

Each instruction is stored using:

- repeater = `1`
- purple wool = `0`

Using the `mcschematic` library, the schematic was parsed and converted into machine code.

This produced:

```text
37 instructions
```

---

## Step 4: Disassemble

Disassembling using the opcode table:

```text
0:  LDI r1 0
1:  LDI r2 55
2:  STR r1 r2 0
3:  ADI r1 1
4:  LDI r2 101
5:  STR r1 r2 0
6:  ADI r1 1
7:  LDI r2 41
8:  STR r1 r2 0
9:  ADI r1 1
10: LDI r2 100
11: STR r1 r2 0
12: ADI r1 1
13: LDI r2 125
14: STR r1 r2 0
15: ADI r1 1
16: LDI r2 57
17: STR r1 r2 0
18: ADI r1 1
19: LDI r2 35
20: STR r1 r2 0
21: ADI r1 1
22: LDI r2 109
23: STR r1 r2 0
24: LDI r1 0
25: LDI r2 4

.loop:
26: LOD r1 r3 0
27: ADI r1 1
28: LOD r1 r4 0
29: ADI r1 1
30: XOR r3 r4 r5
31: LDI r6 250
32: STR r6 r5 0
33: ADI r2 -1
34: BRH eq 36
35: JMP 26
36: HLT
```

---

## Step 5: Compute the Output

The program:

1. Stores values in memory
2. Reads them in pairs
3. Applies XOR
4. Outputs the result

### Computation

| Pair | Values     | XOR |
|------|-----------|-----|
| 1    | 55, 101   | 82  |
| 2    | 41, 100   | 77  |
| 3    | 125, 57   | 68  |
| 4    | 35, 109   | 78  |

### Result

```text
82 77 68 78
```

Concatenated:

```text
82776878
```

---

## Flag

```text
VBD{82776878}
```

Thanks for reading and happy hacking!
