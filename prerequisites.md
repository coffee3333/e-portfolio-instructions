# Prerequisites for coding part

**Before starting, make sure everything on this checklist is in order.**

## 1. Python

You need **Python 3.12 or 3.13**.

Check your version:

```bash
python3 --version
# Expected: Python 3.12.x or 3.13.x
```

> **Warning:** On Windows the command may be `python` or `py` instead of `python3`. Run `py --version` if `python3` is not found. The rest of this guide uses `python3` — substitute accordingly on Windows.

If Python is not installed or the version is too old, download it from [python.org](https://www.python.org/downloads/). Use the latest 3.12 or 3.13 release.

---

## 2. Virtual Environments

A virtual environment keeps this project's dependencies isolated from other Python projects and from your system Python. Always use one.

**Create a virtual environment:**

```bash
python3 -m venv venv
```

**Activate it:**

```bash
# macOS / Linux
source venv/bin/activate

# Windows (Command Prompt)
venv\Scripts\activate.bat

# Windows (PowerShell)
venv\Scripts\Activate.ps1
```

When activated, your prompt gains a `(venv)` prefix:

```
(venv) $
```

**Deactivate** when you're done with a session:

```bash
deactivate
```


> **Warning:** You must activate the virtual environment every time you open a new terminal. If `pip install` or `python manage.py` behaves unexpectedly, the first thing to check is whether `(venv)` is visible in your prompt.

---

## 3. pip

pip is the Python package installer. It is included with Python 3.

Upgrade pip before installing anything else:

```bash
pip install --upgrade pip
```

Verify the upgrade:

```bash
pip --version
# Expected: pip 24.x or higher
```

Common pip commands you will use in this guide:

```bash
pip install <package>           # install a package
pip install <package>==5.2.*    # install a specific version range
pip list                        # show all installed packages
pip freeze > requirements.txt   # save a snapshot of installed packages
pip install -r requirements.txt # install from a saved snapshot
```

---

## 4. Recommended Tools

### Code Editor

- **PyCharm** - Professional edition has Django-specific tooling (recommended)
- **VS Code** - install the [Python extension](https://marketplace.visualstudio.com/items?itemName=ms-python.python)

### API Testing

You will call your API endpoints during development. Use one of:

| Tool | Type | Notes                                                     |
|---|---|-----------------------------------------------------------|
| [Postman](https://www.postman.com/) | GUI app | Most popular, free tier is sufficient                     |
| [Bruno](https://www.usebruno.com/) | GUI app | Open-source, stores collections as files                  |
| curl | Terminal | Built into macOS/Linux; available on Windows via Git Bash |

**curl basics** — the three commands you'll use most:

```bash
# GET request
curl http://127.0.0.1:8000/api/todos/

# POST request with JSON body
curl -X POST http://127.0.0.1:8000/api/todos/ \
  -H "Content-Type: application/json" \
  -d '{"title": "Buy milk"}'

# Request with Authorization header
curl http://127.0.0.1:8000/api/todos/ \
  -H "Authorization: Bearer eyJ..."
```

---

## 5. Common Pitfalls

> **Warning: Windows PATH issues**  
> If `python3` is not found on Windows, make sure "Add Python to PATH" was checked during installation. You can repair the installation and re-check that option.

> **Warning: `python` vs. `python3`**  
> On some systems `python` points to Python 2 and `python3` to Python 3. Always check with `python3 --version` first.

> **Warning: Installing packages globally**  
> Running `pip install` without an active virtual environment installs packages system-wide. This causes version conflicts across projects. Always see `(venv)` in your prompt before running pip.

---

## Checklist

Before the class, confirm:

- `python3 --version` shows 3.12 or 3.13
- You can create and activate a virtual environment
- `pip install --upgrade pip` runs without errors
- You have a code editor installed
- You have curl, Postman, or Bruno available for API testing (Not Necessary)


**Next:** [Project Setup](setup-django.md)
