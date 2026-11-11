You're on the right track with wanting to adopt a **migrator pattern** — it's a more robust and controlled way to manage schema evolution and table creation across services, especially when **referential integrity** is required.

In Python with SQLAlchemy, the typical approach involves:

---

### ✅ Solution Outline

1. **Single Source of Truth for Schema**:

   * Define all your models (tables and relationships) in a single location or shared module (used by both migrator and consuming services).

2. **Migration Tool**: Use [**Alembic**](https://alembic.sqlalchemy.org/) — the de facto migration tool for SQLAlchemy.

   * Alembic handles schema migrations, versioning, and upgrades in a controlled, repeatable way.

3. **Project Structure**:

   * You'd have:

     * `migrator/`: The service that runs Alembic migrations.
     * `service_a/`, `service_b/`: Consumers that only **use** the database schema.
     * `models/`: A shared package for SQLAlchemy model definitions.

---

### 🛠️ Implementation Details

#### 1. Shared Model Definitions (e.g. `models/base.py`)

```python
from sqlalchemy.orm import declarative_base

Base = declarative_base()
```

#### 2. Example Models (e.g. `models/user.py`)

```python
from sqlalchemy import Column, Integer, String
from .base import Base

class User(Base):
    __tablename__ = 'users'

    id = Column(Integer, primary_key=True)
    name = Column(String, nullable=False)
```

> Services A and B would **import these model definitions** to interact with the DB — not define their own.

---

### 3. Set Up Alembic in Migrator

Inside `migrator/`, install Alembic and initialize:

```bash
pip install alembic
cd migrator
alembic init alembic
```

Update `alembic.ini` and `env.py`:

#### `alembic/env.py` (important!)

Modify it to use your shared models:

```python
from logging.config import fileConfig
from sqlalchemy import engine_from_config, pool
from alembic import context
import sys
import os

# Add path to shared models
sys.path.insert(0, os.path.abspath(os.path.join(os.path.dirname(__file__), '..', 'models')))
from base import Base  # Your declarative base
from user import User  # Import all models

config = context.config
fileConfig(config.config_file_name)

target_metadata = Base.metadata  # This is what Alembic uses for migrations

def run_migrations_offline():
    ...

def run_migrations_online():
    ...
```

Now you can auto-generate migrations:

```bash
alembic revision --autogenerate -m "Initial tables"
alembic upgrade head
```

---

### 4. Services A and B Usage

In `service_a/` and `service_b/`, you just import the shared models and interact with the DB like normal:

```python
from sqlalchemy.orm import sessionmaker
from sqlalchemy import create_engine
from models.user import User

engine = create_engine(DB_URL)
Session = sessionmaker(bind=engine)
session = Session()

# Read/write logic
user = session.query(User).first()
```

❗️Avoid re-declaring any models or running `Base.metadata.create_all()` — only the **migrator** should create/modify tables.

---

### ✅ Benefits of This Approach

* Centralized schema definitions.
* Reliable and repeatable migrations.
* Version-controlled schema evolution.
* No race conditions or side effects from simultaneous table creation.

---

### 🧠 Tip: Alembic in Docker Compose

If you're containerizing everything, you can have a `migrator` service in `docker-compose.yml` that waits for the DB and then runs:

```bash
alembic upgrade head
```

Before the main services start.

---

Would you like a **template project layout** for this with Docker and everything scaffolded?


# Details


Great — now we're getting into the **practicalities of containerized code sharing and imports**.

You're already on a good base with:

* Each service in a containerized Dockerfile setup
* Scripts and dependencies in `service_name/app/`
* Wanting to share model definitions (e.g. for SQLAlchemy) **between the migrator and FastAPI services** (like `service_b`)

Let’s walk through a **clean, idiomatic way to share models across services** using Docker, FastAPI, and Alembic.

---

## ✅ GOAL

* Define all SQLAlchemy models in a **shared, importable Python package**
* Use these models:

  * In the `migrator/` service via **Alembic**
  * In services like `service_b/` (e.g. FastAPI) to query the DB
* All services are containerized, and can import the shared models cleanly

---

## ✅ PROJECT STRUCTURE (Recommended)

```
project-root/
│
├── shared/
│   └── models/
│       ├── __init__.py
│       ├── base.py
│       ├── user.py
│       └── ...other models...
│
├── migrator/
│   ├── Dockerfile
│   ├── alembic.ini
│   └── alembic/
│       ├── env.py  ← imports from shared.models
│       └── versions/
│
├── service_b/
│   ├── Dockerfile
│   └── app/
│       ├── main.py
│       └── deps/
│           └── db.py (Session setup, imports shared.models)
│
└── requirements.txt
```

The key is the `shared/` directory. It acts as a **Python package** that can be mounted or installed into each service.

---

## 🔧 HOW TO IMPORT SHARED MODELS IN EACH SERVICE

### Option 1: Mount `shared/` into each container during Docker build

In your `service_b/Dockerfile`:

```dockerfile
FROM python:3.11

WORKDIR /app

# Install requirements
COPY pyproject...  .
RUN pip install --no-cache-dir -r requirements.txt

# Copy app code
COPY service_b/app ./app

# Copy shared models into container
COPY shared /shared

# Set PYTHONPATH to include shared
ENV PYTHONPATH="/shared:/app"

CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

Then in `service_b/app/main.py` or `db.py`:

```python
from shared.models.user import User  # works!
from shared.models.base import Base
```

Do the same in the `migrator` Dockerfile.

---

### Option 2: Package `shared/` as an installable wheel or editable install

This is a bit more complex but useful if you want full decoupling. Let me know if you want this route and I’ll walk you through it.

---

## ✅ DATABASE SESSION (FastAPI Example)

In `service_b/app/deps/db.py`:

```python
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker

DB_URL = "postgresql://user:password@db:5432/mydb"

engine = create_engine(DB_URL)
SessionLocal = sessionmaker(autocommit=False, autoflush=False, bind=engine)
```

In `service_b/app/main.py`:

```python
from fastapi import FastAPI, Depends
from sqlalchemy.orm import Session
from app.deps.db import SessionLocal
from shared.models.user import User

app = FastAPI()

def get_db():
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()

@app.get("/users")
def list_users(db: Session = Depends(get_db)):
    return db.query(User).all()
```

---

## ✅ Alembic Setup (in `migrator/alembic/env.py`)

Just import models like this:

```python
import sys
import os
sys.path.append(os.path.abspath(os.path.join(os.path.dirname(__file__), '../../shared')))

from models.base import Base
from models import user  # import all needed models

target_metadata = Base.metadata
```

---

## ✅ Docker Compose (Optional Integration)

If you're using `docker-compose`, you can mount the `shared/` directory like so:

```yaml
version: "3.9"

services:
  db:
    image: postgres:15
    ...

  migrator:
    build: ./migrator
    volumes:
      - ./shared:/shared
    depends_on:
      - db
    command: alembic upgrade head

  service_b:
    build: ./service_b
    volumes:
      - ./shared:/shared
    depends_on:
      - db
```

---

## ✅ Summary

| Task              | Solution                           |
| ----------------- | ---------------------------------- |
| Share models      | Use `shared/models/` package       |
| Import in FastAPI | Set `PYTHONPATH` or install shared |
| Use in Alembic    | Modify `env.py` to import models   |
| Docker support    | Copy or mount `shared/` directory  |

---

Would you like a GitHub-ready skeleton template for this whole structure? Or help adjusting your current Dockerfiles for this pattern?