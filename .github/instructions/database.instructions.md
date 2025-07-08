---
applyTo: '**'
---

# Integração com Banco de Dados

## Configuração do SQLAlchemy

### Configuração Base
```python
from sqlalchemy import create_engine
from sqlalchemy.ext.declarative import declarative_base
from sqlalchemy.orm import sessionmaker
from sqlalchemy.ext.asyncio import AsyncSession, create_async_engine
from app.core.config import settings

# Engine síncrono
engine = create_engine(
    settings.database_url,
    pool_pre_ping=True,
    pool_recycle=300,
    pool_size=10,
    max_overflow=20
)

# Engine assíncrono
async_engine = create_async_engine(
    settings.async_database_url,
    pool_pre_ping=True,
    pool_recycle=300,
    pool_size=10,
    max_overflow=20
)

SessionLocal = sessionmaker(autocommit=False, autoflush=False, bind=engine)
AsyncSessionLocal = sessionmaker(
    async_engine, class_=AsyncSession, expire_on_commit=False
)

Base = declarative_base()

# Dependência para obter sessão do banco
def get_db():
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()

async def get_async_db():
    async with AsyncSessionLocal() as session:
        yield session
```

### Modelos SQLAlchemy
```python
from sqlalchemy import Column, Integer, String, DateTime, Boolean, ForeignKey, Text
from sqlalchemy.orm import relationship
from sqlalchemy.sql import func
from app.core.database import Base

class User(Base):
    __tablename__ = "users"
    
    id = Column(Integer, primary_key=True, index=True)
    email = Column(String(255), unique=True, index=True, nullable=False)
    name = Column(String(100), nullable=False)
    password_hash = Column(String(255), nullable=False)
    is_active = Column(Boolean, default=True)
    created_at = Column(DateTime(timezone=True), server_default=func.now())
    updated_at = Column(DateTime(timezone=True), onupdate=func.now())
    
    # Relacionamentos
    posts = relationship("Post", back_populates="author")

class Post(Base):
    __tablename__ = "posts"
    
    id = Column(Integer, primary_key=True, index=True)
    title = Column(String(200), nullable=False)
    content = Column(Text, nullable=False)
    author_id = Column(Integer, ForeignKey("users.id"))
    is_published = Column(Boolean, default=False)
    created_at = Column(DateTime(timezone=True), server_default=func.now())
    updated_at = Column(DateTime(timezone=True), onupdate=func.now())
    
    # Relacionamentos
    author = relationship("User", back_populates="posts")
```

## Implementação de Repositórios

### Interface do Repositório
```python
from abc import ABC, abstractmethod
from typing import List, Optional, Generic, TypeVar
from sqlalchemy.orm import Session

T = TypeVar('T')

class BaseRepository(ABC, Generic[T]):
    @abstractmethod
    async def create(self, obj: T) -> T:
        pass
    
    @abstractmethod
    async def get_by_id(self, id: int) -> Optional[T]:
        pass
    
    @abstractmethod
    async def update(self, id: int, obj: T) -> Optional[T]:
        pass
    
    @abstractmethod
    async def delete(self, id: int) -> bool:
        pass
    
    @abstractmethod
    async def list_all(self, skip: int = 0, limit: int = 100) -> List[T]:
        pass
```

### Implementação do Repositório
```python
from sqlalchemy.orm import Session
from sqlalchemy import select, update, delete
from typing import List, Optional
from app.domain.entities.user import User as UserEntity
from app.infrastructure.database.models import User as UserModel
from app.domain.repositories.user_repository import UserRepository

class UserRepositoryImpl(UserRepository):
    def __init__(self, db: Session):
        self.db = db
    
    async def create(self, user: UserEntity) -> UserEntity:
        """Criar usuário no banco de dados."""
        db_user = UserModel(
            email=user.email,
            name=user.name,
            password_hash=user.password_hash
        )
        self.db.add(db_user)
        self.db.commit()
        self.db.refresh(db_user)
        
        return self._to_entity(db_user)
    
    async def get_by_id(self, user_id: int) -> Optional[UserEntity]:
        """Buscar usuário por ID."""
        db_user = self.db.query(UserModel).filter(UserModel.id == user_id).first()
        return self._to_entity(db_user) if db_user else None
    
    async def get_by_email(self, email: str) -> Optional[UserEntity]:
        """Buscar usuário por email."""
        db_user = self.db.query(UserModel).filter(UserModel.email == email).first()
        return self._to_entity(db_user) if db_user else None
    
    async def update(self, user_id: int, user: UserEntity) -> Optional[UserEntity]:
        """Atualizar usuário."""
        db_user = self.db.query(UserModel).filter(UserModel.id == user_id).first()
        if not db_user:
            return None
        
        db_user.email = user.email
        db_user.name = user.name
        if user.password_hash:
            db_user.password_hash = user.password_hash
        
        self.db.commit()
        self.db.refresh(db_user)
        
        return self._to_entity(db_user)
    
    async def delete(self, user_id: int) -> bool:
        """Deletar usuário."""
        db_user = self.db.query(UserModel).filter(UserModel.id == user_id).first()
        if not db_user:
            return False
        
        self.db.delete(db_user)
        self.db.commit()
        return True
    
    async def list_all(self, skip: int = 0, limit: int = 100) -> List[UserEntity]:
        """Listar usuários com paginação."""
        db_users = self.db.query(UserModel).offset(skip).limit(limit).all()
        return [self._to_entity(db_user) for db_user in db_users]
    
    def _to_entity(self, db_user: UserModel) -> UserEntity:
        """Converter modelo do banco para entidade de domínio."""
        return UserEntity(
            id=db_user.id,
            email=db_user.email,
            name=db_user.name,
            password_hash=db_user.password_hash,
            is_active=db_user.is_active,
            created_at=db_user.created_at,
            updated_at=db_user.updated_at
        )
```

## Migrations com Alembic

### Configuração do Alembic
```python
# alembic/env.py
from alembic import context
from sqlalchemy import engine_from_config, pool
from app.core.database import Base
from app.infrastructure.database.models import *  # Importar todos os modelos

config = context.config

# Adicionar Base.metadata
target_metadata = Base.metadata

def run_migrations_online():
    """Run migrations in 'online' mode."""
    connectable = engine_from_config(
        config.get_section(config.config_ini_section),
        prefix="sqlalchemy.",
        poolclass=pool.NullPool,
    )

    with connectable.connect() as connection:
        context.configure(
            connection=connection,
            target_metadata=target_metadata,
            compare_type=True,
            compare_server_default=True,
        )

        with context.begin_transaction():
            context.run_migrations()
```

### Comandos Alembic
```bash
# Inicializar Alembic
alembic init alembic

# Criar nova migração
alembic revision --autogenerate -m "Create users table"

# Aplicar migrações
alembic upgrade head

# Reverter migração
alembic downgrade -1

# Ver histórico
alembic history

# Ver migração atual
alembic current
```

## Queries Complexas

### Queries com Joins
```python
from sqlalchemy.orm import joinedload
from typing import List

class PostRepositoryImpl:
    def __init__(self, db: Session):
        self.db = db
    
    async def get_posts_with_authors(self, skip: int = 0, limit: int = 100) -> List[PostEntity]:
        """Buscar posts com informações do autor."""
        db_posts = (
            self.db.query(PostModel)
            .options(joinedload(PostModel.author))
            .offset(skip)
            .limit(limit)
            .all()
        )
        return [self._to_entity(post) for post in db_posts]
    
    async def get_published_posts_by_author(self, author_id: int) -> List[PostEntity]:
        """Buscar posts publicados de um autor específico."""
        db_posts = (
            self.db.query(PostModel)
            .filter(PostModel.author_id == author_id)
            .filter(PostModel.is_published == True)
            .order_by(PostModel.created_at.desc())
            .all()
        )
        return [self._to_entity(post) for post in db_posts]
    
    async def search_posts(self, query: str, skip: int = 0, limit: int = 100) -> List[PostEntity]:
        """Buscar posts por título ou conteúdo."""
        db_posts = (
            self.db.query(PostModel)
            .filter(
                PostModel.title.ilike(f"%{query}%") |
                PostModel.content.ilike(f"%{query}%")
            )
            .filter(PostModel.is_published == True)
            .offset(skip)
            .limit(limit)
            .all()
        )
        return [self._to_entity(post) for post in db_posts]
```

### Queries com Agregações
```python
from sqlalchemy import func, desc

class AnalyticsRepository:
    def __init__(self, db: Session):
        self.db = db
    
    async def get_user_post_count(self) -> List[dict]:
        """Obter contagem de posts por usuário."""
        results = (
            self.db.query(
                UserModel.id,
                UserModel.name,
                func.count(PostModel.id).label('post_count')
            )
            .outerjoin(PostModel)
            .group_by(UserModel.id, UserModel.name)
            .order_by(desc('post_count'))
            .all()
        )
        
        return [
            {
                'user_id': result.id,
                'user_name': result.name,
                'post_count': result.post_count
            }
            for result in results
        ]
    
    async def get_monthly_post_stats(self) -> List[dict]:
        """Obter estatísticas mensais de posts."""
        results = (
            self.db.query(
                func.date_trunc('month', PostModel.created_at).label('month'),
                func.count(PostModel.id).label('total_posts'),
                func.count(PostModel.id).filter(PostModel.is_published == True).label('published_posts')
            )
            .group_by(func.date_trunc('month', PostModel.created_at))
            .order_by(desc('month'))
            .all()
        )
        
        return [
            {
                'month': result.month,
                'total_posts': result.total_posts,
                'published_posts': result.published_posts
            }
            for result in results
        ]
```

## Transações

### Gerenciamento de Transações
```python
from sqlalchemy.orm import Session
from contextlib import contextmanager

@contextmanager
def db_transaction(db: Session):
    """Context manager para transações."""
    try:
        yield db
        db.commit()
    except Exception as e:
        db.rollback()
        raise e

class UserService:
    def __init__(self, user_repository: UserRepository, post_repository: PostRepository):
        self.user_repository = user_repository
        self.post_repository = post_repository
    
    async def create_user_with_welcome_post(self, user: UserEntity) -> UserEntity:
        """Criar usuário com post de boas-vindas em transação."""
        with db_transaction(self.user_repository.db):
            # Criar usuário
            created_user = await self.user_repository.create(user)
            
            # Criar post de boas-vindas
            welcome_post = PostEntity(
                title="Bem-vindo!",
                content="Obrigado por se juntar à nossa plataforma!",
                author_id=created_user.id,
                is_published=True
            )
            await self.post_repository.create(welcome_post)
            
            return created_user
```

## Conexão com Múltiplos Bancos

### Configuração Multi-Database
```python
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker

# Banco principal
primary_engine = create_engine(settings.primary_database_url)
PrimarySessionLocal = sessionmaker(autocommit=False, autoflush=False, bind=primary_engine)

# Banco de análise
analytics_engine = create_engine(settings.analytics_database_url)
AnalyticsSessionLocal = sessionmaker(autocommit=False, autoflush=False, bind=analytics_engine)

# Dependências
def get_primary_db():
    db = PrimarySessionLocal()
    try:
        yield db
    finally:
        db.close()

def get_analytics_db():
    db = AnalyticsSessionLocal()
    try:
        yield db
    finally:
        db.close()
```

## Performance e Otimização

### Indexação
```python
from sqlalchemy import Index

class User(Base):
    __tablename__ = "users"
    
    id = Column(Integer, primary_key=True, index=True)
    email = Column(String(255), unique=True, nullable=False)
    name = Column(String(100), nullable=False)
    created_at = Column(DateTime(timezone=True), server_default=func.now())
    
    # Índices compostos
    __table_args__ = (
        Index('idx_user_email_active', 'email', 'is_active'),
        Index('idx_user_created_at', 'created_at'),
    )
```

### Eager Loading
```python
from sqlalchemy.orm import selectinload, joinedload

class PostRepositoryImpl:
    async def get_posts_with_all_relations(self) -> List[PostEntity]:
        """Buscar posts com todas as relações carregadas."""
        db_posts = (
            self.db.query(PostModel)
            .options(
                joinedload(PostModel.author),
                selectinload(PostModel.comments).joinedload(CommentModel.author)
            )
            .all()
        )
        return [self._to_entity(post) for post in db_posts]
```

## Boas Práticas

1. **Separation of Concerns**: Mantenha lógica de banco separada da lógica de negócio
2. **Transactions**: Use transações para operações que devem ser atômicas
3. **Connection Pooling**: Configure pool de conexões adequadamente
4. **Indexing**: Crie índices para queries frequentes
5. **Lazy Loading**: Use eager loading quando necessário
6. **Migrations**: Use Alembic para controle de versão do schema
7. **Error Handling**: Trate erros de banco adequadamente
8. **Performance**: Monitore e otimize queries lentas
9. **Security**: Use prepared statements para evitar SQL injection
10. **Testing**: Teste repositórios com banco de dados de teste
