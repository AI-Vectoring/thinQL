# thinQL
LLM-native SQL generation tooling for Go.

## 🧠 Phloisophy

* **Minimalism for humans, pragmatism for agents.**
* **Make your LLMs happy: Your request lives in the same file as your business logic.**

thinQL is a streamlined SQL-first library built for LLM-driven and agentic development.
No ORM. No config. No annotations.
Just SQL → typed data.

* **You describe your logic in SQL, you get back typed Go.**
Everytrhing else is abstracted away... bye, bye!
Pure control, zero boilerplate

---

## 🔧 Setup

A schema generator is included. You don’t write types — thinQL infers them once and uses them at runtime.

---

## 🚀 Usage

### Query with Struct (Only Way You’ll Ever Need)

```go
type User struct {
    ID   int    `db:"id"`
    Name string `db:"name"`
}

var user User
err := thinql.Query("SELECT id, name FROM users WHERE id = 1", &user)
```

Write SQL. Get data. All statically typed, all in one block.

Perfect for production, amazing for coding agents.


---

## ✅ Why thinQL

* ✅ **SQL as the source of truth**
* ✅ **Zero boilerplate, obvious intent**
* ✅ **Safe through schema inference**
* ✅ **Fits LLMs like a glove**
* ✅ **Built for automation, scales to real use**

---

## 🔮 The Future

thinQL is built for the rise of **agentic LLM development** — systems where agents reason, act, and adapt.

Let your LLMs generate SQL. Let thinQL do the rest.

Keep your business logic and data access in sync — with nothing lost in translation.

---

## 📦 Installation

```sh
go get github.com/yourorg/thinql
```

---

## 🧪 License

MIT
