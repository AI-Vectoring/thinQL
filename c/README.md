#                         thinQL — C
##        Tooling for LLM-native SQL generation in C.


## Philosophy

* **Minimalism for humans, pragmatism for agents.**
* **Make your LLMs happy: Your request lives in the same file as your business logic.**

thinQL is a streamlined SQL-first library built for LLM-driven and agentic development.
No ORM. No config. No annotations.
Just SQL → typed data.

---

## Installation

```sh
# via package or build from source
make && make install
```

---

## Usage

A schema generator is included. You don't write types — thinQL infers them once and uses them at runtime.

```c
#include "thinql.h"

typedef struct {
    int  id;
    char name[256];
} User;

User user;
thinql_query("SELECT id, name FROM users WHERE id = 1", &user, THINQL_TYPE(User));
```

Write SQL. Get data. All statically typed, all in one block.
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
