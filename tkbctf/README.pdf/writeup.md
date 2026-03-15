# README.pdf Write-up

Challenge: `README.pdf`  
Category: `Rev`  
Points: `90`


## 1. Open the PDF and inspect what it does

At first glance, the PDF looks like a simple flag checker with clickable buttons for letters, digits, `{`, `}`, and `_`.
That already suggests the file might contain embedded logic instead of being a normal static document.

## 2. Extract readable strings from the PDF
![PDF page](assets/01_pdf_page.png)
PDF files often contain JavaScript in plain text or near-plain text. A fast first check is to run `strings` and search for suspicious logic.

```bash
strings -n 6 '/home/kali/readme (1).pdf' | rg -n -C 4 'var expected|var k = 90|function check_flag|result_field.value = "Correct"'
```

The important part is:

```javascript
var expected = [46,49,56,57,46,60,33,16,110,44,110,9,57,40,107,42,46,5,107,52,5,10,30,28,39];
var k = 90;

function check_flag() {
    ...
    if ((input_buf.charCodeAt(i) ^ k) !== expected[i]) {
        result_field.value = "Wrong";
        return;
    }
    result_field.value = "Correct";
}
```

This tells us the checker is doing:

`input_char ^ 90 == expected_value`

So to recover the real input, we just reverse it:

`input_char = expected_value ^ 90`

## 3. Decode the flag

A short Python snippet is enough:

```python
expected = [46, 49, 56, 57, 46, 60, 33, 16, 110, 44, 110, 9, 57, 40, 107, 42, 46, 5, 107, 52, 5, 10, 30, 28, 39]
k = 90
flag = ''.join(chr(x ^ k) for x in expected)
print(flag)
```


Output:

```text
tkbctf{J4v4Scr1pt_1n_PDF}
```

## 4. Final Flag

tkbctf{J4v4Scr1pt_1n_PDF}

Thanks for reading and happy hacking!


4. XOR every value in `expected` with `90`.
5. The decoded result is the flag.
