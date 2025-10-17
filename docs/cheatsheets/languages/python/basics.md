---
title: Python Basics
tags:
    - python
    - programming
    - cheatsheet
    - basics
summary: A compact, beginner-friendly overview of Python fundamentals.
aliases:
  - Python Basics Cheatsheet
---

# 🐍 Python Basics Cheatsheet

---

## 🧠 What Python Is

Python is a **high-level**, **interpreted** programming language known for readability and simplicity.
It focuses on *clarity over cleverness*. You write `.py` files and run them directly — no manual compilation needed.

It’s widely used for **web backends (Django, Flask, FastAPI)**, **automation**, **data science**, **AI/ML**, and **scripting**.

```bash
python --version       # check Python version
python main.py         # run script
```

---

## ⚙️ Setup and Environment

```bash
# Create and activate virtual environment
python -m venv .venv
source .venv/bin/activate      # Linux/Mac
# .venv\Scripts\activate       # Windows

# Install packages
pip install requests
```

Use `venv` to isolate dependencies per project.
This keeps versions clean and avoids global conflicts.

---

## 🧱 Python Basics

### Variables & Data Types

No type declarations — Python is **dynamically typed**.

```python
x = 42
pi = 3.14
name = "Edgaras"
is_ready = True
```

| Type    | Example          | Description        |
| ------- | ---------------- | ------------------ |
| `int`   | `10`             | Whole number       |
| `float` | `3.14`           | Decimal number     |
| `str`   | `"hello"`        | Text               |
| `bool`  | `True` / `False` | Logical value      |
| `list`  | `[1, 2, 3]`      | Ordered, mutable   |
| `tuple` | `(1, 2, 3)`      | Ordered, immutable |
| `set`   | `{1, 2, 3}`      | Unique, unordered  |
| `dict`  | `{"a": 1}`       | Key-value pairs    |

---

### Printing and Comments

```python
print("Hello, world!")    # output text

# Single-line comment
"""
Multi-line
comment
"""
```

---

## 🔁 Control Flow

```python
if x > 10:
    print("Big")
elif x > 5:
    print("Medium")
else:
    print("Small")

for fruit in ["apple", "banana"]:
    print(fruit)

while x < 5:
    x += 1
```

Use indentation instead of braces — Python uses **whitespace to define scope**.

---

## 🔧 Functions

Functions are defined with `def`, return values with `return`.

```python
def greet(name="friend"):
    return f"Hello, {name}!"

print(greet("Edgaras"))
```

Short functions (lambdas):

```python
add = lambda a, b: a + b
```

---

## 🧩 Collections in Action

Lists, sets, tuples, and dicts are the backbone of Python.

```python
# List
nums = [1, 2, 3]
nums.append(4)

# Tuple
coords = (10, 20)

# Set
unique = {1, 2, 2, 3}  # {1, 2, 3}

# Dict
user = {"name": "Alice", "age": 25}
print(user["name"])
```

List comprehension (Pythonic pattern):

```python
squares = [x*x for x in range(5)]  # [0, 1, 4, 9, 16]
```

---

## 📁 Files and Paths

```python
with open("data.txt", "r") as f:
    text = f.read()

with open("output.txt", "w") as f:
    f.write("Hello file!")
```

Modern way:

```python
from pathlib import Path
data = Path("data.txt").read_text()
```

---

## 🚨 Errors and Exceptions

Handle problems gracefully:

```python
try:
    risky()
except ValueError as e:
    print("Error:", e)
finally:
    print("Cleanup done")
```

Raise your own error:

```python
raise Exception("Something went wrong")
```

---

## 🧱 Classes and Objects

Python supports **object-oriented programming**, but doesn’t force it.

```python
class Dog:
    def __init__(self, name):
        self.name = name

    def bark(self):
        print(f"{self.name} says woof!")

dog = Dog("Max")
dog.bark()
```

Everything is an object — even functions, classes, and modules.

---

## 📦 Modules & Imports

Split code across files and reuse it.

```python
import math
from datetime import date

print(math.sqrt(16))
print(date.today())
```

Run logic only when executed directly:

```python
if __name__ == "__main__":
    main()
```

---

## ✨ Modern Python Features

```python
# F-strings (formatted strings)
user = "Alice"
print(f"Hi, {user}!")

# Type hints (optional, for clarity)
def add(x: int, y: int) -> int:
    return x + y

# List comprehension
squares = [n*n for n in range(5)]

# Pattern matching (Python 3.10+)
match command:
    case "start":
        print("Running...")
    case "stop":
        print("Stopped")
    case _:
        print("Unknown")
```

---

## 🧰 Common Tools

| Tool              | Purpose                   |
| ----------------- | ------------------------- |
| `pip`             | Install packages          |
| `venv`            | Virtual environments      |
| `pytest`          | Testing                   |
| `black`, `flake8` | Code formatting & linting |
| `jupyter`         | Interactive notebooks     |
| `pathlib`         | Modern path handling      |
| `requests`        | Simple HTTP client        |

---

## 🧭 The Python Mindset

* Prefer **readable** over clever.
* Use built-ins before reinventing logic.
* Code should explain itself — less ceremony, more clarity.
* Experiment freely in the REPL (`python` in terminal).

> “Simple is better than complex.”
> “Readability counts.”
> — *The Zen of Python* (`import this`)

