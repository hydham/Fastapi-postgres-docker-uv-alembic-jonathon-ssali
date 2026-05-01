# FastAPI + Postgres Dev Environment (Practical Beginner Guide — Jonathan’s Flow)

Hi everyone, Jonathan here. In this guide we’ll set up a **development environment** for a FastAPI app that uses **Postgres** as the database, with **SQLAlchemy + Alembic** for database migrations, and **Docker Compose** to run everything together. We’ll also use **Docker Compose watch** so code changes on your laptop automatically sync into the running container.

The lecture uses **uv** (and `uvx`) for Python project + dependency management. I’ll follow that same flow.  
Then, because beginners often meet other standards in teams, I’ll also add “from my side” how to do the **same kickstart** with **pip**, **pipenv**, and **poetry**.

If you follow this guide end-to-end, you’ll get:

- A FastAPI app running on `http://localhost:8000`
- A Postgres DB running in Docker
- A persistent DB volume (data survives restarts)
- A clean config setup (env vars in `.env`)
- A DB connection that runs during app startup (lifespan)
- A simple SQLAlchemy model created in the DB
- Docker Compose watch (sync + rebuild)
- Alembic migrations that work **inside** Docker (important!)

---

## Chapter 1 — What we’re building (mental picture)

We’ll run **two containers**:

1. **app**: your FastAPI application (built from a Dockerfile)
2. **db**: Postgres (official image)

Docker Compose will:

- create a private network
- start both containers on that network
- let the app reach the DB using the hostname `db` (service name)
- store Postgres data in a volume so it doesn’t disappear when containers restart

---

## Chapter 2 — Create a new FastAPI project (uv + uvx)

Jonathan uses `uvx` to run a “FastAPI new project” command.

Think of `uvx` as:  
“Run a Python CLI tool (from a package) quickly, without you manually installing it globally.”

### 2.1 Create project
Pick a folder name (example: `fastapi-sqlalchemy-docker`).

```bash
uvx fastapi new fastapi-sqlalchemy-docker
cd fastapi-sqlalchemy-docker
code .
```

What you should see in VS Code:

- `.venv/` — uv created virtual environment
- `main.py` — basic FastAPI app + one endpoint
- `pyproject.toml` — project metadata + dependencies
- `README.md` — run instructions
- `uv.lock` — locked dependency versions

### 2.2 Run the default app (sanity check)
```bash
uv run fastapi dev
```

Open:
- `http://localhost:8000`  
You should see the default “Hello World” style response.

Stop it with Ctrl+C.

---

## Chapter 3 — Dockerize the app (Dockerfile + .dockerignore)

We want the app to run inside a container, and we want uv available inside that container.

Jonathan uses a uv-provided image: **uv Debian slim**.

### 3.1 Dockerfile (explained line-by-line)

Create `Dockerfile` in the project root:

```dockerfile
# Base image that already contains uv
FROM ghcr.io/astral-sh/uv:debian-slim

# All project files will live here inside the container
WORKDIR /app

# Copy only dependency files first (better Docker caching)
COPY pyproject.toml uv.lock ./

# Install exactly what the lock file says
RUN uv sync

# Copy the rest of the code
COPY . .

# FastAPI dev server uses 8000 by default
EXPOSE 8000

# Important: listen on 0.0.0.0 so the app is reachable from outside the container
CMD ["uv", "run", "fastapi", "dev", "--host", "0.0.0.0", "--port", "8000"]
```

Why `0.0.0.0`?  
Inside a container, `127.0.0.1` means “only inside that container”.  
To reach the app from your laptop, the app must listen on **all interfaces**: `0.0.0.0`.

### 3.2 .dockerignore (keep the image clean)

Create `.dockerignore`:

```
.venv
__pycache__
```

This is like `.gitignore`, but for Docker build context.  
We don’t want to copy virtualenvs or Python cache files into the image.

---

## Chapter 4 — Docker Compose: run app + Postgres together

Create `compose.yml` (or `docker-compose.yml`). Jonathan calls it a “compose file”.

### 4.1 Minimal compose.yml (two services + volume)

Create `compose.yml`:

```yaml
services:
  app:
    build: .
    ports:
      - "8000:8000"

  db:
    image: postgres:16
    environment:
      POSTGRES_USER: user
      POSTGRES_PASSWORD: password
      POSTGRES_DB: mydatabase
    volumes:
      - db-data:/var/lib/postgresql/data

volumes:
  db-data:
```

Explanation (beginner language):

- `services:` is the list of containers we want.
- `app:` is built from your Dockerfile.
- `ports: "8000:8000"` maps container port 8000 → laptop port 8000.
- `db:` runs Postgres image.
- `environment:` tells Postgres what user/password/db to create on first startup.
- `volumes:` persists Postgres data. Without it, your DB resets every time.

### 4.2 Run it
Run this in the folder that contains `compose.yml`:

```bash
docker compose up --build
```

Open:
- `http://localhost:8000`

Stop with Ctrl+C.

---

## Chapter 5 — Add real dependencies (asyncpg + sqlalchemy + pydantic-settings)

Now we add the packages Jonathan adds:

```bash
uv add asyncpg sqlalchemy pydantic-settings
uv lock
```

Why `uv lock`?  
It updates `uv.lock` with the exact dependency versions so Docker can reproduce them.

Now rebuild containers so the new deps get installed:

```bash
docker compose up --build
```

Stop with Ctrl+C.

---

## Chapter 6 — Move DB credentials to .env (so compose.yml isn’t hardcoded)

Create `.env`:

```
POSTGRES_USER=user
POSTGRES_PASSWORD=password
POSTGRES_DB=mydatabase
```

Now update `compose.yml` to read env vars:

```yaml
services:
  app:
    build: .
    ports:
      - "8000:8000"

  db:
    image: postgres:16
    environment:
      POSTGRES_USER: ${POSTGRES_USER}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      POSTGRES_DB: ${POSTGRES_DB}
    volumes:
      - db-data:/var/lib/postgresql/data

volumes:
  db-data:
```

Important: compose automatically loads `.env` when it’s in the same folder.

---

## Chapter 7 — Create a clean Python structure + settings loader

Jonathan creates a `source/` folder for project code.

### 7.1 Create package structure

Create:

- `source/`
- `source/__init__.py`
- `source/config.py`

### 7.2 config.py (pydantic-settings)

`source/config.py`:

```python
from pydantic_settings import BaseSettings, SettingsConfigDict


class Settings(BaseSettings):
    POSTGRES_USER: str
    POSTGRES_PASSWORD: str
    POSTGRES_DB: str

    model_config = SettingsConfigDict(
        env_file=".env",
        env_file_encoding="utf-8",
        extra="ignore",  # ignore unknown keys in .env
    )


config = Settings()
```

Beginner explanation:

- `Settings` reads environment variables and validates them.
- `config = Settings()` creates the settings object you’ll import elsewhere.

---

## Chapter 8 — DB connection: async engine + init_db()

Create:

- `source/db/`
- `source/db/__init__.py`
- `source/db/connection.py`

### 8.1 connection.py (create_async_engine + test query)

`source/db/connection.py`:

```python
from sqlalchemy.ext.asyncio import create_async_engine
from sqlalchemy import text

from source.config import config

DB_URL = (
    f"postgresql+asyncpg://{config.POSTGRES_USER}:{config.POSTGRES_PASSWORD}"
    f"@db:5432/{config.POSTGRES_DB}"
)

engine = create_async_engine(DB_URL, echo=True)


async def init_db() -> None:
    async with engine.begin() as conn:
        stmt = text("SELECT 'database initialized' AS message;")
        result = await conn.execute(stmt)
        print(result.all())
```

Key beginner points:

- Hostname is **`db`**, not `localhost` (Docker DNS uses service name).
- `echo=True` prints SQL logs so you can see what’s happening.

---

## Chapter 9 — Run init_db during FastAPI startup (lifespan)

Open `main.py` and wire lifespan.

```python
from fastapi import FastAPI
from contextlib import asynccontextmanager

from source.db.connection import init_db


@asynccontextmanager
async def lifespan(app: FastAPI):
    print("server is starting")
    await init_db()
    yield
    print("server is stopping")


app = FastAPI(lifespan=lifespan)


@app.get("/")
async def root():
    return {"message": "hello world"}
```

Run:

```bash
docker compose up --build
```

You should see printed result like `[('database initialized',)]`.

---

## Chapter 10 — Create a simple SQLAlchemy model + create tables

Create `source/db/models.py`:

```python
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column
from sqlalchemy import String, Integer


class Base(DeclarativeBase):
    pass


class User(Base):
    __tablename__ = "o_users"

    id: Mapped[int] = mapped_column(Integer, primary_key=True)
    username: Mapped[str] = mapped_column(String(100), nullable=False)
    hashed_password: Mapped[str] = mapped_column(String(200), nullable=False)
```

Update `init_db()` to create tables:

```python
from source.db.models import Base

async def init_db() -> None:
    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.create_all)
        print("tables created")
```

Rebuild + run:

```bash
docker compose up --build
```

---

## Chapter 11 — Verify table inside Postgres (docker compose exec)

Run detached:

```bash
docker compose up -d
```

Enter psql:

```bash
docker compose exec -it db psql -U user -d mydatabase
```

Inside psql:

- list tables: `\dt`
- describe: `\d o_users`
- query: `SELECT * FROM o_users;`

Exit: `\q`

---

## Chapter 12 — Docker Compose watch (sync + rebuild)

Add to `compose.yml` under `app`:

```yaml
services:
  app:
    build: .
    ports:
      - "8000:8000"
    develop:
      watch:
        - action: sync
          path: .
          target: /app
        - action: rebuild
          path: pyproject.toml
        - action: ignore
          path: .venv
```

Run:

```bash
docker compose up --build
```

Press **W** to enable watch.

Now code changes sync into `/app`, and FastAPI dev reload will respond.

---

## Chapter 13 — Alembic migrations (and why they must run inside Docker)

### 13.1 Install Alembic
```bash
uv add alembic
uv lock
```

### 13.2 List templates
```bash
uv run alembic list_templates
```

### 13.3 Init migrations (pyproject + async template)
```bash
uv run alembic init --template pyproject_async migrations
```

### 13.4 Configure migrations/env.py (URL + metadata)

We don’t want to hardcode DB URL in `alembic.ini`. Instead we set it programmatically.

In `migrations/env.py`:

- build DB_URL using your `config`
- set it with `set_main_option`
- set `target_metadata = Base.metadata`

Example snippet:

```python
from source.config import config as app_config
from source.db.models import Base

db_url = (
    f"postgresql+asyncpg://{app_config.POSTGRES_USER}:{app_config.POSTGRES_PASSWORD}"
    f"@db:5432/{app_config.POSTGRES_DB}"
)

context.config.set_main_option("sqlalchemy.url", db_url)
target_metadata = Base.metadata
```

### 13.5 The lecture’s key “gotcha”
If you run Alembic **on your laptop**, it can’t resolve `db` because `db` only exists inside the Docker network.

So: exec into the app container and run Alembic there.

```bash
docker compose exec -it app bash
```

Inside container:

```bash
uv run alembic revision --autogenerate -m "add created_at datetime field"
uv run alembic upgrade head
```

Exit: `exit`

Verify in Postgres that the new column exists.

---

## Chapter 14 — Common fixes (the ones that actually save you)

### If Postgres env vars were wrong earlier
Postgres initialization is stored in the volume. If you fix `.env` later, the old volume may keep the old DB state.

Reset everything:

```bash
docker compose down -v
docker compose up --build
```

### If disk space is low
```bash
docker system prune
```

---

# Appendix — Same kickstart with pip, pipenv, poetry (fills the gaps)

Even though Jonathan uses uv, here’s how you’d start the same kind of FastAPI project in other ecosystems.

## pip + venv
```bash
mkdir demo-project
cd demo-project
python -m venv .venv
source .venv/bin/activate

pip install fastapi uvicorn[standard]
pip install sqlalchemy asyncpg pydantic-settings alembic
```

Run:
```bash
uvicorn main:app --reload --host 127.0.0.1 --port 8000
```

Lock (basic):
```bash
pip freeze > requirements.txt
```

## pipenv
```bash
mkdir demo-project
cd demo-project
pip install pipenv

pipenv install fastapi uvicorn[standard]
pipenv install sqlalchemy asyncpg pydantic-settings alembic

pipenv run uvicorn main:app --reload --host 127.0.0.1 --port 8000
```

Lock is automatic: `Pipfile.lock`.

## poetry
```bash
mkdir demo-project
cd demo-project
poetry init -n

poetry add fastapi uvicorn[standard]
poetry add sqlalchemy asyncpg pydantic-settings alembic

poetry run uvicorn main:app --reload --host 127.0.0.1 --port 8000
```

Lock is automatic: `poetry.lock`.

---

## Final quick commands (copy/paste)

Build + run:
```bash
docker compose up --build
```

Stop:
```bash
docker compose down
```

Reset DB volume:
```bash
docker compose down -v
```

Exec into db:
```bash
docker compose exec -it db psql -U user -d mydatabase
```

Exec into app:
```bash
docker compose exec -it app bash
```
