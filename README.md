# Easy Snake

**CTF:** SPARK CTF  
**Category:** Reverse Engineering  
**Difficulty:** Easy  
**Flag:** `Spark{7h3_sn4k3_is_34sy_righ7?}`

---

## Description

We are given a Linux binary called `easy_snake`. Running it tells us:

```
Usage: ./easy_snake <your_flag>
```

The goal is to figure out the correct flag by reversing the binary.

---

## Solution

### Step 1 - Identify the Binary

The first thing to do is check what kind of binary we are dealing with:

```bash
file easy_snake
```

Output:

```
easy_snake: ELF 64-bit LSB executable, x86-64, dynamically linked ...
```

Running `strings` or looking at the file size (~8 MB for a simple flag checker) and the presence of strings like `_MEIPASS`, `pyi-`, or `pyiboot` are strong hints that this was packed with **PyInstaller**.

### Step 2 - Extract the PyInstaller Archive

PyInstaller bundles a Python interpreter, all dependencies, and your script into a single binary. The tool [pyinstxtractor](https://github.com/extremecoders-re/pyinstxtractor) can unpack it.

First, download it:

```bash
curl -sL https://raw.githubusercontent.com/extremecoders-re/pyinstxtractor/master/pyinstxtractor.py -o pyinstxtractor.py
```

Then run it using the **same Python version** that was used to build the binary. Running `strings easy_snake | grep "python"` or just trying the extraction and reading the output tells us the binary was built with **Python 3.13**.

```bash
python3.13 pyinstxtractor.py easy_snake
```

Output:

```
[+] Processing easy_snake
[+] Pyinstaller version: 2.1+
[+] Python version: 3.13
[+] Length of package: 8119261 bytes
[+] Found 30 files in CArchive
[+] Beginning extraction...please standby
[+] Possible entry point: easy_snake.pyc
[+] Found 112 files in PYZ archive
[+] Successfully extracted pyinstaller archive: easy_snake
```

> **Why use the matching Python version?** PyInstaller compresses the PYZ archive using the marshal format, which is version-specific. Using a different Python version causes the PYZ extraction to be skipped entirely.

The extracted directory contains several files, the most important being:

```
easy_snake_extracted/
    easy_snake.pyc        <-- our target
    PYZ.pyz_extracted/    <-- bundled stdlib modules
    ...
```

### Step 3 - Decompile the .pyc File

A `.pyc` file is a compiled Python bytecode file. We have several options to read it:

- **pycdc** (C++ decompiler, best output)
- **decompyle3 / uncompyle6** (Python decompilers)
- **Python's built-in `dis` module** (always works, no install needed)

Since this is Python 3.13, many decompilers do not support it yet. The `dis` module is the most reliable fallback:

```bash
python3.13 -c "
import dis, marshal

with open('easy_snake_extracted/easy_snake.pyc', 'rb') as f:
    f.read(16)  # skip header (magic + flags + timestamp + size)
    code = marshal.loads(f.read())

dis.dis(code)
"
```

### Step 4 - Read the Bytecode

The disassembly output shows two code objects: the module-level code and a function called `check_flag`. The interesting part is inside `check_flag`:

```
  4   BUILD_LIST               0
      LOAD_CONST               1 ((83, 112, 97, 114, 107, 123, 55, 104, 51, 95,
                                   115, 110, 52, 107, 51, 95, 105, 115, 95, 51,
                                   52, 115, 121, 95, 114, 105, 103, 104, 55, 63, 125))
      LIST_EXTEND              1
      STORE_FAST               1 (magic)
```

The variable `magic` holds a tuple of integers. The function then iterates over the input string and compares `ord(s[i])` against `magic[i]` for each character. This means `magic` **is** the flag encoded as ASCII ordinals.

### Step 5 - Decode the Flag

```python
values = (83, 112, 97, 114, 107, 123, 55, 104, 51, 95, 115, 110,
          52, 107, 51, 95, 105, 115, 95, 51, 52, 115, 121, 95,
          114, 105, 103, 104, 55, 63, 125)

print(''.join(chr(c) for c in values))
```

Output:

```
Spark{7h3_sn4k3_is_34sy_righ7?}
```

### Verification

```bash
./easy_snake 'Spark{7h3_sn4k3_is_34sy_righ7?}'
# Correct! You got the flag!
```

---

## Summary

| Step | Action |
|------|--------|
| 1 | Recognize the binary as a PyInstaller bundle |
| 2 | Extract with `pyinstxtractor` using Python 3.13 |
| 3 | Locate `easy_snake.pyc` as the entry point |
| 4 | Disassemble with Python's `dis` module |
| 5 | Read the hardcoded `magic` tuple from bytecode |
| 6 | Decode ASCII ordinals to get the flag |

**Key takeaway:** PyInstaller does not protect your source code. The compiled `.pyc` bytecode is trivially reversible and any constants (including hardcoded flags) are fully visible in the disassembly.

---

## Tools Used

- [pyinstxtractor](https://github.com/extremecoders-re/pyinstxtractor) - PyInstaller archive extractor
- Python 3.13 built-in `dis` and `marshal` modules

---

## Author

**becem69 😝**

Challenge designed, built, and solved by becem69 for SPARK CTF.  
If you have questions, feedback, or just want to talk reversing, feel free to reach out.
