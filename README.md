# TP Nro 6 - Desarrollo de Sistemas Backend - Tutorial FastAPI

## Paso 1: Crear y activar entorno virtual

`python3 -m venv env`

Linux/macOS:
`source env/bin/activate`

Windows:
`env/Scripts/activate`


## Paso 2: Instalar dependencias

`pip install fastapi uvicorn sqlalchemy aiosqlite alembic`

Guardar dependencias:
`pip freeze > requirements.txt`


## Paso 3: Estructura de carpetas sugerida

Podés crear esta estructura con los comandos que prefieras o desde tu editor:

```
app/
├── main.py
├── database.py
├── models.py
├── schemas.py
├── dal.py
├── routers/
│ ├── __init__.py
│ ├── usuarios.py
│ └── alumnos.py
alembic/ # Esto se crea solo!
├── env.py
├── versions/
requirements.txt
```


## Paso 4: App básica "Hola mundo"

```python 
# app/main.py

from fastapi import FastAPI
from app.routers import usuarios, alumnos
from app.database import Base, engine

app = FastAPI()

app.include_router(usuarios.router, prefix="/usuarios", tags=["usuarios"])
app.include_router(alumnos.router, prefix="/alumnos", tags=["alumnos"])

@app.get("/")
async def hola_mundo():
  return {"mensaje": "Hola mundo"}
```


## Paso 5: Configurar la base de datos

```python 
# app/database.py

from sqlalchemy.ext.asyncio import create_async_engine, AsyncSession
from sqlalchemy.orm import sessionmaker, declarative_base

DATABASE_URL = "sqlite+aiosqlite:///./database.db"

engine = create_async_engine(DATABASE_URL, echo=True)

AsyncSessionLocal = sessionmaker(
  bind=engine,
  class_=AsyncSession,
  expire_on_commit=False
)

Base = declarative_base()
target_metadata = Base.metadata
```


## Paso 6: Modelos con relaciones

```python
# app/models.py

from sqlalchemy import Column, Integer, String, ForeignKey
from sqlalchemy.orm import relationship
from .database import Base

class Usuario(Base):
  __tablename__ = "usuario"
  id = Column(Integer, primary_key=True, index=True)
  nombre = Column(String, nullable=True)
  email = Column(String, unique=True, index=True, nullable=False)
  alumno = relationship("Alumno", back_populates="usuario", uselist=False)

class Alumno(Base):
  __tablename__ = "alumno"
  id = Column(Integer, primary_key=True, index=True)
  carrera = Column(String, nullable=True)
  usuario_id = Column(Integer, ForeignKey("usuario.id"))
  usuario = relationship("Usuario", back_populates="alumno")
```


## Paso 7: Schemas (request/response)

```python
# app/schemas.py

from pydantic import BaseModel
from typing import Optional

# REQUEST

class UsuarioCreateRequest(BaseModel):
  nombre: Optional[str] = None
  email: str

class AlumnoCreateRequest(BaseModel):
  carrera: Optional[str] = None
  usuario_id: int


# RESPONSE

class UsuarioResponse(BaseModel):
  id: int
  nombre: Optional[str]
  email: str

  class Config:
    orm_mode = True

class AlumnoResponse(BaseModel):
  id: int
  carrera: Optional[str]
  usuario: UsuarioResponse
  
  class Config:
    orm_mode = True
```


## Paso 8: Lógica DAL

```python 
# app/dal.py

from sqlalchemy.ext.asyncio import AsyncSession
from sqlalchemy.future import select
from sqlalchemy.orm import selectinload
from app import models, schemas

async def crear_usuario(db: AsyncSession, usuario: schemas.UsuarioCreateRequest):
  nuevo_usuario = models.Usuario(**usuario.dict())
  db.add(nuevo_usuario)
  await db.commit()
  await db.refresh(nuevo_usuario)
  return nuevo_usuario

async def crear_alumno(db: AsyncSession, alumno: schemas.AlumnoCreateRequest):
  nuevo_alumno = models.Alumno(**alumno.dict())
  db.add(nuevo_alumno)
  await db.commit()
  await db.refresh(nuevo_alumno)
  result = await db.execute(
    select(models.Alumno)
    .options(selectinload(models.Alumno.usuario))
    .where(models.Alumno.id == nuevo_alumno.id)
  )

  alumno_con_relacion = result.scalar_one()
  return alumno_con_relacion

async def obtener_usuarios(db: AsyncSession):
  result = await db.execute(select(models.Usuario))
  return result.scalars().all()

async def obtener_alumnos(db: AsyncSession):
  result = await db.execute(
  select(models.Alumno).options(
    selectinload(models.Alumno.usuario))
  )
  return result.scalars().all()
```


## Paso 10: Migraciones con Alembic

1. Inicializar Alembic

- Fuera de la carepta `app/`, en la carpeta del proyecto:
`alembic init alembic`


2. Configurar `alembic.ini`
`sqlalchemy.url = sqlite:///./database.db`

> Importante: no usar `aiosqlite` en Alembic. Solo `sqlite:///` sin async.


3. Editar `alembic/env.py`

- Al comienzo del archivo, agregar:
```python
from app.database import Base
from app import models
```

- Luego, reemplazar esta línea:
```python
target_metadata = None 
```

por:
```python
target_metadata = Base.metadata
```


4. Crear y aplicar migración

`alembic revision --autogenerate -m "create usuario y alumno"`

`alembic upgrade head`


## Listo

¡Ya tenés una API funcional con FastAPI, SQLite, SQLAlchemy, Alembic y endpoints REST!

Podés levantar un servidor, dentro de la carpeta de su proyecto con:

`uvicorn app.main:app --reload`

Accedé a la documentación de OpenAPI en:

http://localhost:8000/docs
