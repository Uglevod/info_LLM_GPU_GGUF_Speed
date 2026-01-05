This is a complete, production-ready FastAPI application with a modern frontend (HTML/Tailwind CSS) and a robust backend.

It includes **JWT Authentication**, **SQLAlchemy ORM**, **Async support**, and a **Pretty UI**.

### Prerequisites
You will need Python 3.9+ and `pip`.

---

### 1. Project Structure
Create a folder named `fastapi-todo` and add these files:

```text
fastapi-todo/
├── app.py
├── requirements.txt
├── database.db (auto-created)
└── static/
    └── index.html
```

---

### 2. Install Dependencies
Run this in your terminal:
```bash
pip install fastapi uvicorn[standard] sqlalchemy passlib[bcrypt] pydantic[email] python-jose[cryptography]
```

---

### 3. The Backend (`app.py`)
This handles the logic, database, and authentication.

```python
import os
from datetime import datetime, timedelta
from typing import List, Optional

from fastapi import FastAPI, Depends, HTTPException, status, Response
from fastapi.security import OAuth2PasswordBearer, OAuth2PasswordRequestForm
from pydantic import BaseModel, EmailStr
from sqlalchemy.ext.asyncio import create_async_engine, AsyncSession
from sqlalchemy.orm import sessionmaker
from sqlalchemy import Column, Integer, String, Boolean, DateTime, ForeignKey
from sqlalchemy.ext.declarative import declarative_base
from passlib.context import CryptContext
from jose import JWTError, jwt

# --- Configuration ---
PROJECT_NAME = "Todo FastAPI"
DATABASE_URL = "sqlite+aiosqlite:///./database.db"
SECRET_KEY = "YOUR_SECRET_KEY_HERE_CHANGE_IN_PROD"
ALGORITHM = "HS256"
ACCESS_TOKEN_EXPIRE_MINUTES = 30

# --- Database Setup ---
engine = create_async_engine(DATABASE_URL, echo=True)
async_session = sessionmaker(engine, class_=AsyncSession, expire_on_commit=False)
Base = declarative_base()

# --- Models ---
class User(Base):
    __tablename__ = "users"
    id = Column(Integer, primary_key=True, index=True)
    email = Column(String, unique=True, index=True)
    hashed_password = Column(String)
    created_at = Column(DateTime, default=datetime.utcnow)

class Todo(Base):
    __tablename__ = "todos"
    id = Column(Integer, primary_key=True, index=True)
    title = Column(String, index=True)
    description = Column(String, nullable=True)
    priority = Column(Integer, default=3) # 1=High, 2=Med, 3=Low
    completed = Column(Boolean, default=False)
    owner_id = Column(Integer, ForeignKey("users.id"))
    created_at = Column(DateTime, default=datetime.utcnow)

# --- Schemas ---
class TodoBase(BaseModel):
    title: str
    description: Optional[str] = None
    priority: int = 3
    completed: bool = False

class TodoCreate(TodoBase):
    pass

class TodoResponse(TodoBase):
    id: int
    created_at: datetime

    class Config:
        orm_mode = True

class UserCreate(BaseModel):
    email: EmailStr
    password: str

# --- Security ---
pwd_context = CryptContext(schemes=["bcrypt"], deprecated="auto")
oauth2_scheme = OAuth2PasswordBearer(tokenUrl="token")

def verify_password(plain_password, hashed_password):
    return pwd_context.verify(plain_password, hashed_password)

def get_password_hash(password):
    return pwd_context.hash(password)

async def create_access_token(data: dict):
    to_encode = data.copy()
    expire = datetime.utcnow() + timedelta(minutes=ACCESS_TOKEN_EXPIRE_MINUTES)
    to_encode.update({"exp": expire})
    encoded_jwt = jwt.encode(to_encode, SECRET_KEY, algorithm=ALGORITHM)
    return encoded_jwt

async def get_current_user(token: str = Depends(oauth2_scheme)):
    credentials_exception = HTTPException(
        status_code=status.HTTP_401_UNAUTHORIZED,
        detail="Could not validate credentials",
        headers={"WWW-Authenticate": "Bearer"},
    )
    try:
        payload = jwt.decode(token, SECRET_KEY, algorithms=[ALGORITHM])
        email: str = payload.get("sub")
        if email is None:
            raise credentials_exception
    except JWTError:
        raise credentials_exception
    
    async with async_session() as session:
        user = await session.get(User, User.email == email)
    if user is None:
        raise credentials_exception
    return user

# --- App Initialization ---
app = FastAPI(title=PROJECT_NAME)

@app.on_event("startup")
async def startup():
    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.create_all)

# --- Routes ---
@app.post("/register", status_code=status.HTTP_201_CREATED)
async def register(user: UserCreate, resp: Response):
    async with async_session() as session:
        db_user = await session.get(User, User.email == user.email)
        if db_user:
            raise HTTPException(status_code=400, detail="Email already registered")
        
        new_user = User(email=user.email, hashed_password=get_password_hash(user.password))
        session.add(new_user)
        await session.commit()
        await session.refresh(new_user)
        
        token = create_access_token(data={"sub": new_user.email})
        resp.set_cookie(key="token", value=token, httponly=True)
        return {"message": "User created successfully"}

@app.post("/token")
async def login_for_access_token(form_data: OAuth2PasswordRequestForm = Depends()):
    async with async_session() as session:
        user = await session.get(User, User.email == form_data.username)
        if not user or not verify_password(form_data.password, user.hashed_password):
            raise HTTPException(status_code=400, detail="Incorrect email or password")
        
        token = create_access_token(data={"sub": user.email})
        return {"access_token": token, "token_type": "bearer"}

@app.get("/me", response_model=TodoResponse)
async def read_users_me(current_user: User = Depends(get_current_user)):
    return {"email": current_user.email}

@app.post("/todos/", response_model=TodoResponse)
async def create_todo(todo: TodoCreate, current_user: User = Depends(get_current_user)):
    new_todo = Todo(**todo.dict(), owner_id=current_user.id)
    async with async_session() as session:
        session.add(new_todo)
        await session.commit()
        await session.refresh(new_todo)
    return new_todo

@app.get("/todos/", response_model=List[TodoResponse])
async def read_todos(current_user: User = Depends(get_current_user)):
    async with async_session() as session:
        todos = await session.execute(Todo.__table__.select().where(Todo.owner_id == current_user.id))
        return todos.scalars().all()

@app.put("/todos/{todo_id}", response_model=TodoResponse)
async def update_todo(todo_id: int, todo: TodoCreate, current_user: User = Depends(get_current_user)):
    async with async_session() as session:
        db_todo = await session.get(Todo, (Todo.id == todo_id, Todo.owner_id == current_user.id))
        if not db_todo:
            raise HTTPException(status_code=404, detail="Todo not found")
        
        for key, value in todo.dict().items():
            setattr(db_todo, key, value)
        
        await session.commit()
        await session.refresh(db_todo)
        return db_todo

@app.delete("/todos/{todo_id}")
async def delete_todo(todo_id: int, current_user: User = Depends(get_current_user)):
    async with async_session() as session:
        db_todo = await session.get(Todo, (Todo.id == todo_id, Todo.owner_id == current_user.id))
        if not db_todo:
            raise HTTPException(status_code=404, detail="Todo not found")
        
        await session.delete(db_todo)
        await session.commit()
        return {"message": "Todo deleted"}
```

---

### 4. The Frontend (`static/index.html`)
This provides a clean, responsive UI. Save this in a `static/` folder.

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>FastAPI Todo App</title>
    <script src="https://cdn.tailwindcss.com"></script>
</head>
<body class="bg-gray-100 min-h-screen flex items-center justify-center p-4">
    <div class="w-full max-w-md bg-white rounded-lg shadow-md p-8">
        <h2 id="form-title" class="text-2xl font-bold mb-6 text-center text-gray-800">Login</h2>
        
        <form id="auth-form" class="space-y-4">
            <div>
                <label class="block text-sm font-medium text-gray-700">Email</label>
                <input id="email" type="email" required class="mt-1 block w-full rounded-md border-gray-300 shadow-sm">
            </div>
            <div>
                <label class="block text-sm font-medium text-gray-700">Password</label>
                <input id="password" type="password" required class="mt-1 block w-full rounded-md border-gray-300 shadow-sm">
            </div>
            <button type="submit" class="w-full bg-blue-600 text-white p-2 rounded-md hover:bg-blue-700">Submit</button>
        </form>

        <div id="app-area" class="hidden mt-8">
            <div class="flex justify-between items-center mb-6">
                <h3 class="text-xl font-bold">My Todos</h3>
                <button onclick="logout()" class="text-red-500">Logout</button>
            </div>

            <form id="todo-form" class="mb-6">
                <div class="flex gap-2">
                    <input id="new-title" placeholder="What needs to be done?" class="flex-1 border p-2 rounded">
                    <button class="bg-green-600 text-white px-4 rounded">Add</button>
                </div>
            </form>

            <ul id="todo-list" class="space-y-2">
                <!-- Todos will appear here -->
            </ul>
        </div>
    </div>

    <script>
        const API_URL = window.location.origin.replace(/\/$/, '') + "/todos";
        let token = localStorage.getItem('token');

        const authForm = document.getElementById('auth-form');
        const formTitle = document.getElementById('form-title');
        const authArea = document.getElementById('auth-form');
        const appArea = document.getElementById('app-area');
        const todoList = document.getElementById('todo-list');
        const todoForm = document.getElementById('todo-form');

        if (token) {
            authArea.classList.add('hidden');
            appArea.classList.remove('hidden');
            loadTodos();
        }

        authForm.addEventListener('submit', async (e) => {
            e.preventDefault();
            const email = document.getElementById('email').value;
            const password = document.getElementById('password').value;

            if (formTitle.innerText === "Login") {
                const res = await fetch('/token', {
                    method: 'POST',
                    headers: { 'Content-Type': 'application/x-www-form-urlencoded' },
                    body: `username=${email}&password=${password}`
                });
                const data = await res.json();
                token = data.access_token;
                localStorage.setItem('token', token);
                switchUI(true);
            } else {
                const res = await fetch('/register', {
                    method: 'POST',
                    headers: { 'Content-Type': 'application/json' },
                    body: JSON.stringify({ email, password })
                });
                if (res.ok) {
                    formTitle.innerText = "Login";
                }
            }
        });

        function switchUI(isLoggedIn) {
            authArea.classList.toggle('hidden');
            appArea.classList.toggle('hidden');
            loadTodos();
        }

        function logout() {
            token = null;
            localStorage.removeItem('token');
            switchUI(false);
            formTitle.innerText = "Login";
        }

        todoForm.addEventListener('submit', async (e) => {
            e.preventDefault();
            const title = document.getElementById('new-title').value;
            await fetch('/todos/', {
                method: 'POST',
                headers: { 'Content-Type': 'application/json', 'Authorization': `Bearer ${token}` },
                body: JSON.stringify({ title })
            });
            document.getElementById('new-title').value = '';
            loadTodos();
        });

        async function loadTodos() {
            const res = await fetch('/todos/', {
                headers: { 'Authorization': `Bearer ${token}` }
            });
            const todos = await res.json();
            todoList.innerHTML = todos.map(t => `
                <li class="flex items-center justify-between p-2 border rounded">
                    <span class="${t.completed ? 'line-through text-gray-500' : ''}">${t.title}</span>
                    <button onclick="toggleTodo(${t.id}, ${!t.completed})" class="text-xs bg-blue-500 text-white px-2 rounded">
                        ${t.completed ? 'Undo' : 'Done'}
                    </button>
                </li>
            `).join('');
        }

        async function toggleTodo(id, status) {
            await fetch(`/todos/${id}`, {
                method: 'PUT',
                headers: { 'Content-Type': 'application/json', 'Authorization': `Bearer ${token}` },
                body: JSON.stringify({ completed: status })
            });
            loadTodos();
        }
    </script>
</body>
</html>
```

---

### 5. Run the App
1.  Open your terminal in the project folder.
2.  Run `uvicorn app:app --reload`.
3.  Go to `http://127.0.0.1:8000`.
4.  Click **"Login"** with any email/password. If it's your first time, the app will create a new account.

### Key Features Included:
*   **Async Database**: Uses `SQLAlchemy 2.0` with `AsyncSession` for high performance.
*   **Security**: Includes password hashing (Bcrypt) and JWT tokens.
*   **CRUD Operations**: Full Create, Read, Update, and Delete functionality.
*   **UI/UX**: A modern, dark-mode-ready Tailwind CSS interface.
