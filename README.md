# Prems

A minimal Flask project scaffold using SQLAlchemy, Alembic, Pydantic, and HTTPX.

## Local startup

```powershell
py -3.13 -m venv .venv
.\.venv\Scripts\Activate.ps1
python -m pip install --upgrade pip
python -m pip install -e ".[dev]"
Copy-Item .env.example .env
$env:FLASK_APP = "prems.app:create_app"
flask run --debug
```

Run checks with:

```powershell
pytest
ruff check .
black --check .
mypy src
```
