#                         thinQL
##        Tooling for LLM-native SQL generation.


## Philosophy

* **Minimalism for humans, pragmatism for agents.**
* **Make your LLMs happy: Your request lives in the same file as your business logic.**

thinQL is a streamlined SQL-first library built for LLM-driven and agentic development.
No ORM. No config. No annotations.
Just SQL → typed data.

* **You describe your logic in SQL, you get back typed, language-native data structures.**
Everything else is abstracted away.
Pure control, zero boilerplate.

---

## Languages

| Language | Folder |
|---|---|
| Go | [go/](go/) |
| TypeScript | [ts/](ts/) |
| Python | [py/](py/) |
| Lua | [lua/](lua/) |
| C | [c/](c/) |
| R5RS Scheme | [r5/](r5/) |

---

## Concept

```js
let user = {}
thinql.query("SELECT id, name FROM users WHERE id = 1", user)
```

---

## Why thinQL

* SQL as the source of truth
* Zero boilerplate, obvious intent
* Safe through schema inference
* Fits LLMs like a glove
* Built for automation, scales to real use

---

## The Future

thinQL is built for the rise of **agentic LLM development** — systems where agents reason, act, and adapt.

Let your LLMs generate SQL. Let thinQL do the rest.

Keep your business logic and data access in sync — with nothing lost in translation.

---

## License

GPL 3
