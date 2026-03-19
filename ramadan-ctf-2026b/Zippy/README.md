# Zippy Write-up

## Challenge Info

- **Name:** Zippy  
- **Category:** Forensics  
- **Points:** 75  
- **Difficulty:** Easy  
- **Author:** VBD  

---

## Description

How much data is lost during compression?  
dont keep the lock and key at the same place

**File:** `locked_files.rar`

---

## Step 1: Initial Analysis

The file is very small (~357 bytes), which indicates hidden data rather than normal compression.

Hex dump:

```text
00000120: 00 03 53 54 4D 10 07 3A 66 6F 72 67 6F 74 70 61
00000130: 73 73 77 6F 72 64 C4 BA 24 44 03 23 F8 40 42 52
```

Visible string:

```text
:forgotpassword
```

`STM` suggests the presence of **NTFS Alternate Data Streams (ADS)**.

---

## Step 2: Extract RAR

Use the discovered password:

```bash
7z x -pforgotpassword locked_files.rar
```

Output:

```text
locked_files.zip
```

ZIP is encrypted.

---

## Step 3: Understand the Hint

```text
How much data is lost during compression?
```

ZIP is **lossless**, so the password must be hidden outside normal file data.

---

## Step 4: Extract Hidden Stream (ADS)

On Windows:

```powershell
Get-Item locked_files.zip -Stream *
```

Output:

```text
Stream         Length
------         ------
:$DATA            203
forgotpassword     32
```

Read the hidden stream:

```powershell
Get-Content locked_files.zip -Stream forgotpassword
```

Result:

```text
8d364896e034aabe3fc9fd2e05fb1cbe
```

---

## Step 5: Extract ZIP

```bash
7z x -p8d364896e034aabe3fc9fd2e05fb1cbe locked_files.zip
```

---

## Flag

```text
VBD{c99a11a53a3748269e3f86d7ac38df11}
```

---

## Key Takeaways

- RAR5 supports NTFS ADS  
- ADS is hidden on Linux  
- Always inspect small archives  
- Compression doesn’t remove data  
- Hidden streams can store secrets  

---

## Tools

- 7z  
- PowerShell  
- xxd / hexdump  
