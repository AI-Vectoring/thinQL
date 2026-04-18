#                         thinQL — Python
##        Tooling for LLM-native SQL generation in Python.


## Philosophy

* **Minimalism for humans, pragmatism for agents.**
* **Make your LLMs happy: Your request lives in the same file as your business logic.**

thinQL is a streamlined SQL-first library built for LLM-driven and agentic development.
No ORM. No config. No annotations.
Just SQL → typed data.

---

## Installation

```sh
pip install thinql
```

---

## Usage

A schema generator is included. You don't write types — thinQL infers them once and uses them at runtime.

```python
from thinql import query
from dataclasses import dataclass

@dataclass
class User:
    id: int
    name: str

user: User = query("SELECT id, name FROM users WHERE id = 1", User)
```

Write SQL. Get data. All typed, all in one block.
Perfect for production, amazing for coding agents.

---

## Why thinQL

* SQL as the source of truth
* Zero boilerplate, obvious intent
* Safe through schema inference
* Fits LLMs like a glove
* Built for automation, scales to real use

---

## License

GPL 3
