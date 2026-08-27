---
title: "Web Frameworks"
weight: 11
---

# Web Frameworks

Python powers some of the most popular web frameworks. This section covers the three main options — FastAPI for modern async APIs, Flask for lightweight applications, and Django for full-featured projects — and helps you choose between them.

---

## The Python Web Ecosystem

| Framework | Type | Best For | Async |
|-----------|------|----------|-------|
| **FastAPI** | Micro | APIs, microservices | Native (ASGI) |
| **Flask** | Micro | Simple apps, prototypes | Via extensions |
| **Django** | Full-stack | Content sites, admin-heavy apps | Partial (Django 4.1+) |
| Starlette | Micro | FastAPI's foundation | Native (ASGI) |
| Litestar | Micro | FastAPI alternative | Native (ASGI) |

### WSGI vs ASGI

| | WSGI | ASGI |
|-|------|------|
| Model | Synchronous | Asynchronous |
| Concurrency | Thread-per-request | Event loop + async |
| Servers | Gunicorn, uWSGI | Uvicorn, Hypercorn |
| Frameworks | Flask, Django (traditional) | FastAPI, Starlette, Django (channels) |
| WebSockets | Not supported | Supported |

---

## FastAPI

Modern, fast, type-safe API framework built on Starlette and Pydantic:

```python
from fastapi import FastAPI, HTTPException, Depends
from pydantic import BaseModel, EmailStr

app = FastAPI(title="User API", version="1.0.0")

class UserCreate(BaseModel):
    name: str
    email: EmailStr
    age: int | None = None

class UserResponse(BaseModel):
    id: int
    name: str
    email: str

@app.get("/users/{user_id}", response_model=UserResponse)
async def get_user(user_id: int):
    user = await db.find_user(user_id)
    if not user:
        raise HTTPException(status_code=404, detail="User not found")
    return user

@app.post("/users", response_model=UserResponse, status_code=201)
async def create_user(user: UserCreate):
    return await db.create_user(user)

@app.get("/users", response_model=list[UserResponse])
async def list_users(skip: int = 0, limit: int = 20):
    return await db.list_users(skip=skip, limit=limit)
```

### Key Features

| Feature | How |
|---------|-----|
| Auto-generated OpenAPI docs | `/docs` (Swagger UI), `/redoc` |
| Request validation | Pydantic models — automatic 422 on invalid input |
| Dependency injection | `Depends()` for DB sessions, auth, etc. |
| Async native | `async def` endpoints with `await` |
| Type-checked | Editor autocomplete, mypy compatible |

### Dependency Injection

```python
from fastapi import Depends
from sqlalchemy.ext.asyncio import AsyncSession

async def get_db() -> AsyncGenerator[AsyncSession, None]:
    async with async_session() as session:
        yield session

@app.get("/users/{id}")
async def get_user(id: int, db: AsyncSession = Depends(get_db)):
    return await db.get(User, id)
```

---

## Flask

Lightweight, flexible, minimal opinions:

```python
from flask import Flask, jsonify, request, abort

app = Flask(__name__)

@app.route("/users/<int:user_id>", methods=["GET"])
def get_user(user_id):
    user = db.find_user(user_id)
    if not user:
        abort(404)
    return jsonify(user.to_dict())

@app.route("/users", methods=["POST"])
def create_user():
    data = request.get_json()
    user = db.create_user(data["name"], data["email"])
    return jsonify(user.to_dict()), 201

if __name__ == "__main__":
    app.run(debug=True)
```

### Flask Ecosystem

| Extension | Purpose |
|-----------|---------|
| Flask-SQLAlchemy | ORM integration |
| Flask-Migrate | Database migrations (Alembic) |
| Flask-Login | Session-based authentication |
| Flask-CORS | Cross-Origin Resource Sharing |
| Flask-RESTful | REST API utilities |
| Flask-Limiter | Rate limiting |

---

## Django

Full-featured "batteries included" framework:

```python
# models.py
from django.db import models

class User(models.Model):
    name = models.CharField(max_length=100)
    email = models.EmailField(unique=True)
    created_at = models.DateTimeField(auto_now_add=True)

# views.py
from django.http import JsonResponse
from django.views import View

class UserDetailView(View):
    def get(self, request, user_id):
        user = User.objects.get(id=user_id)
        return JsonResponse({"id": user.id, "name": user.name})

# urls.py
urlpatterns = [
    path("users/<int:user_id>/", UserDetailView.as_view()),
]
```

### Django's Batteries

| Feature | Built-in? |
|---------|-----------|
| ORM | ✓ (Django ORM) |
| Admin interface | ✓ (auto-generated CRUD UI) |
| Authentication | ✓ (users, groups, permissions) |
| Migrations | ✓ (`manage.py makemigrations`) |
| Forms / validation | ✓ |
| Template engine | ✓ |
| CSRF protection | ✓ |
| Testing client | ✓ |
| Django REST Framework | Separate package (DRF) for APIs |

---

## Choosing a Framework

| Criterion | FastAPI | Flask | Django |
|-----------|---------|-------|--------|
| API-only service | ✓✓✓ | ✓✓ | ✓ (with DRF) |
| Full web application | ✓ | ✓✓ | ✓✓✓ |
| Learning curve | Low | Low | Medium |
| Admin interface | — | — | ✓✓✓ |
| Async support | Native | Limited | Partial |
| Auto documentation | ✓✓✓ (OpenAPI) | Manual | Manual (DRF has it) |
| Type safety | ✓✓✓ | — | — |
| Community size | Growing fast | Large | Very large |
| Deployment | Uvicorn | Gunicorn | Gunicorn / Daphne |

**Quick decision:**
- Building a JSON API → **FastAPI**
- Quick prototype or microservice → **Flask**
- Full application with admin, auth, ORM → **Django**

---

## Key Takeaways

- FastAPI is the modern default for Python APIs — auto-validation, auto-docs, native async, type-safe.
- Flask is minimal and flexible — good for prototypes and small services. Add extensions as needed.
- Django is "batteries included" — ORM, admin, auth, migrations built in. Ideal for content-heavy applications.
- ASGI (Uvicorn) for async frameworks; WSGI (Gunicorn) for sync. FastAPI requires ASGI.
- Pydantic models in FastAPI serve as request validation, response serialisation, and documentation in one.
- All three frameworks are production-ready. Choose based on your project needs, not hype.
