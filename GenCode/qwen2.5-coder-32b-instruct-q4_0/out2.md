Certainly! Below is a complete example of a FastAPI Todo application. This application will include CRUD (Create, Read, Update, Delete) operations, and we'll use SQLite for simplicity. We'll also include some basic validation and error handling. Additionally, I'll include a simple HTML frontend using Jinja2 templates to make it more user-friendly.

### Step 1: Set Up Your Environment

First, ensure you have FastAPI and Uvicorn installed. You can also install SQLite and Jinja2 for the database and templating, respectively.

```bash
pip install fastapi uvicorn sqlalchemy databases jinja2
```

### Step 2: Create the FastAPI Application

#### Directory Structure

```
/todo_app
    /templates
        index.html
    main.py
    models.py
    schemas.py
    database.py
```

#### `database.py`

This file will handle the database connection.

```python
from databases import Database
from sqlalchemy import create_engine, MetaData

DATABASE_URL = "sqlite:///./test.db"

database = Database(DATABASE_URL)
metadata = MetaData()

engine = create_engine(DATABASE_URL)
metadata.create_all(engine)
```

#### `models.py`

This file defines the SQLAlchemy model.

```python
from sqlalchemy import Table, Column, Integer, String, Boolean
from .database import metadata

todos = Table(
    "todos",
    metadata,
    Column("id", Integer, primary_key=True),
    Column("title", String, nullable=False),
    Column("description", String),
    Column("completed", Boolean, default=False),
)
```

#### `schemas.py`

This file defines Pydantic schemas for data validation.

```python
from pydantic import BaseModel

class TodoBase(BaseModel):
    title: str
    description: str = None

class TodoCreate(TodoBase):
    pass

class Todo(TodoBase):
    id: int
    completed: bool

    class Config:
        orm_mode = True
```

#### `main.py`

This is the main FastAPI application file.

```python
from fastapi import FastAPI, HTTPException, Request
from fastapi.responses import HTMLResponse
from fastapi.staticfiles import StaticFiles
from fastapi.templating import Jinja2Templates
from sqlalchemy import select
from .database import database, engine, metadata, todos
from .schemas import Todo, TodoCreate
from typing import List

app = FastAPI()
app.mount("/static", StaticFiles(directory="static"), name="static")
templates = Jinja2Templates(directory="templates")

@app.on_event("startup")
async def startup():
    await database.connect()

@app.on_event("shutdown")
async def shutdown():
    await database.disconnect()

@app.get("/", response_class=HTMLResponse)
async def read_root(request: Request):
    query = select(todos)
    results = await database.fetch_all(query)
    return templates.TemplateResponse("index.html", {"request": request, "todos": results})

@app.post("/todos/", response_model=Todo)
async def create_todo(todo: TodoCreate):
    query = todos.insert().values(title=todo.title, description=todo.description)
    last_record_id = await database.execute(query)
    return {**todo.dict(), "id": last_record_id, "completed": False}

@app.get("/todos/", response_model=List[Todo])
async def read_todos():
    query = select(todos)
    return await database.fetch_all(query)

@app.get("/todos/{todo_id}", response_model=Todo)
async def read_todo(todo_id: int):
    query = select(todos).where(todos.c.id == todo_id)
    todo = await database.fetch_one(query)
    if not todo:
        raise HTTPException(status_code=404, detail="Todo not found")
    return todo

@app.put("/todos/{todo_id}", response_model=Todo)
async def update_todo(todo_id: int, todo: TodoCreate):
    query = (
        todos
        .update()
        .where(todos.c.id == todo_id)
        .values(title=todo.title, description=todo.description)
        .returning(todos)
    )
    updated_todo = await database.fetch_one(query)
    if not updated_todo:
        raise HTTPException(status_code=404, detail="Todo not found")
    return updated_todo

@app.patch("/todos/{todo_id}/complete", response_model=Todo)
async def complete_todo(todo_id: int):
    query = (
        todos
        .update()
        .where(todos.c.id == todo_id)
        .values(completed=True)
        .returning(todos)
    )
    completed_todo = await database.fetch_one(query)
    if not completed_todo:
        raise HTTPException(status_code=404, detail="Todo not found")
    return completed_todo

@app.delete("/todos/{todo_id}", response_model=Todo)
async def delete_todo(todo_id: int):
    query = todos.delete().where(todos.c.id == todo_id).returning(todos)
    deleted_todo = await database.fetch_one(query)
    if not deleted_todo:
        raise HTTPException(status_code=404, detail="Todo not found")
    return deleted_todo
```

#### `templates/index.html`

This is a simple HTML template to display the todos.

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Todo App</title>
    <link href="https://cdn.jsdelivr.net/npm/bootstrap@5.1.3/dist/css/bootstrap.min.css" rel="stylesheet">
</head>
<body>
    <div class="container">
        <h1>Todo List</h1>
        <form action="/todos/" method="post">
            <div class="mb-3">
                <label for="title" class="form-label">Title</label>
                <input type="text" class="form-control" id="title" name="title" required>
            </div>
            <div class="mb-3">
                <label for="description" class="form-label">Description</label>
                <textarea class="form-control" id="description" name="description"></textarea>
            </div>
            <button type="submit" class="btn btn-primary">Add Todo</button>
        </form>
        <hr>
        <ul class="list-group">
            {% for todo in todos %}
            <li class="list-group-item">
                <div class="d-flex justify-content-between align-items-center">
                    <div>
                        <h5>{{ todo.title }}</h5>
                        <p>{{ todo.description }}</p>
                    </div>
                    <div>
                        {% if not todo.completed %}
                        <form action="/todos/{{ todo.id }}/complete" method="post" style="display:inline;">
                            <button type="submit" class="btn btn-success">Complete</button>
                        </form>
                        {% endif %}
                        <form action="/todos/{{ todo.id }}" method="post" style="display:inline;">
                            <button type="submit" class="btn btn-danger">Delete</button>
                        </form>
                    </div>
                </div>
            </li>
            {% endfor %}
        </ul>
    </div>
    <script src="https://cdn.jsdelivr.net/npm/bootstrap@5.1.3/dist/js/bootstrap.bundle.min.js"></script>
</body>
</html>
```

### Step 3: Run the Application

Run the FastAPI application using Uvicorn:

```bash
uvicorn main:app --reload
```

Visit `http://127.0.0.1:8000` in your browser to see the Todo app in action.

### Explanation

- **FastAPI**: Handles HTTP requests and responses.
- **SQLAlchemy**: ORM for defining database models.
- **Databases**: Asynchronous database driver.
- **Jinja2**: Templating engine for rendering HTML.
- **SQLite**: In-memory database for simplicity.

This example provides a basic structure for a Todo app. You can expand it by adding more features, such as user authentication, more complex validation, and a more sophisticated frontend.
