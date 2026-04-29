# My Website Docs

As the name suggests, it's personal website.

---

Install dependencies and create the project environment:

```
uv sync
```

Local dev server:

```
uv run zensical serve --dev-addr 127.0.0.1:8000
```

Build static site:

```
uv run zensical build --clean
```