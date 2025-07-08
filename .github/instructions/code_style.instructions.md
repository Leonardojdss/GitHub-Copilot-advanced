---
applyTo: '**'
---

# Padrões de Código para FastAPI

## Convenções Gerais

### Nomenclatura
- **Variáveis e funções**: snake_case
- **Classes**: PascalCase
- **Constantes**: UPPER_CASE
- **Módulos**: lowercase com underscores
- **Arquivos**: snake_case.py

### Estrutura de Arquivos
```python
# Ordem de imports
# 1. Bibliotecas padrão do Python
import os
from typing import List, Optional
from datetime import datetime

# 2. Bibliotecas de terceiros
from fastapi import FastAPI, HTTPException, Depends
from pydantic import BaseModel
from sqlalchemy.orm import Session

# 3. Imports locais
from app.core.config import settings
from app.domain.entities.user import User
from app.domain.services.user_service import UserService
```

## Padrões para Endpoints

### Estrutura de Rotas
```python
from fastapi import APIRouter, Depends, HTTPException, status
from typing import List

from app.presentation.schemas.user_schema import UserCreate, UserResponse
from app.domain.services.user_service import UserService
from app.presentation.api.dependencies import get_user_service

router = APIRouter(prefix="/users", tags=["users"])

@router.post("/", response_model=UserResponse, status_code=status.HTTP_201_CREATED)
async def create_user(
    user_data: UserCreate,
    user_service: UserService = Depends(get_user_service)
) -> UserResponse:
    """
    Criar um novo usuário.
    
    Args:
        user_data: Dados do usuário a ser criado
        user_service: Serviço de usuário injetado
    
    Returns:
        UserResponse: Dados do usuário criado
    
    Raises:
        HTTPException: Erro de validação ou conflito
    """
    try:
        user = await user_service.create_user(user_data.to_entity())
        return UserResponse.from_entity(user)
    except ValueError as e:
        raise HTTPException(
            status_code=status.HTTP_400_BAD_REQUEST,
            detail=str(e)
        )

@router.get("/{user_id}", response_model=UserResponse)
async def get_user(
    user_id: int,
    user_service: UserService = Depends(get_user_service)
) -> UserResponse:
    """Buscar usuário por ID."""
    user = await user_service.get_user_by_id(user_id)
    if not user:
        raise HTTPException(
            status_code=status.HTTP_404_NOT_FOUND,
            detail="Usuário não encontrado"
        )
    return UserResponse.from_entity(user)

@router.get("/", response_model=List[UserResponse])
async def list_users(
    skip: int = 0,
    limit: int = 100,
    user_service: UserService = Depends(get_user_service)
) -> List[UserResponse]:
    """Listar usuários com paginação."""
    users = await user_service.list_users(skip=skip, limit=limit)
    return [UserResponse.from_entity(user) for user in users]
```

## Padrões para Schemas (Pydantic)

### Modelos de Entrada e Saída
```python
from pydantic import BaseModel, EmailStr, validator
from typing import Optional
from datetime import datetime

class UserBase(BaseModel):
    email: EmailStr
    name: str
    
    @validator('name')
    def validate_name(cls, v):
        if len(v.strip()) < 2:
            raise ValueError('Nome deve ter pelo menos 2 caracteres')
        return v.strip()

class UserCreate(UserBase):
    password: str
    
    @validator('password')
    def validate_password(cls, v):
        if len(v) < 8:
            raise ValueError('Senha deve ter pelo menos 8 caracteres')
        return v
    
    def to_entity(self) -> User:
        """Converter para entidade de domínio."""
        return User(
            email=self.email,
            name=self.name,
            password=self.password
        )

class UserUpdate(BaseModel):
    name: Optional[str] = None
    email: Optional[EmailStr] = None
    
    @validator('name')
    def validate_name(cls, v):
        if v and len(v.strip()) < 2:
            raise ValueError('Nome deve ter pelo menos 2 caracteres')
        return v.strip() if v else v

class UserResponse(BaseModel):
    id: int
    email: EmailStr
    name: str
    created_at: datetime
    
    class Config:
        from_attributes = True
    
    @classmethod
    def from_entity(cls, user: User) -> 'UserResponse':
        """Criar resposta a partir de entidade de domínio."""
        return cls(
            id=user.id,
            email=user.email,
            name=user.name,
            created_at=user.created_at
        )
```

## Padrões para Dependências

### Injeção de Dependências
```python
from fastapi import Depends
from sqlalchemy.orm import Session

from app.core.database import get_db
from app.infrastructure.database.user_repository_impl import UserRepositoryImpl
from app.domain.services.user_service import UserService

def get_user_repository(db: Session = Depends(get_db)) -> UserRepositoryImpl:
    """Obter repositório de usuário."""
    return UserRepositoryImpl(db)

def get_user_service(
    user_repository: UserRepositoryImpl = Depends(get_user_repository)
) -> UserService:
    """Obter serviço de usuário."""
    return UserService(user_repository)
```

## Tratamento de Erros

### Padronização de Exceções
```python
from fastapi import HTTPException, status
from app.core.exceptions import BusinessException, ValidationException

async def handle_business_exceptions(func):
    """Decorator para tratar exceções de negócio."""
    try:
        return await func()
    except ValidationException as e:
        raise HTTPException(
            status_code=status.HTTP_400_BAD_REQUEST,
            detail=str(e)
        )
    except BusinessException as e:
        raise HTTPException(
            status_code=status.HTTP_422_UNPROCESSABLE_ENTITY,
            detail=str(e)
        )
    except Exception as e:
        raise HTTPException(
            status_code=status.HTTP_500_INTERNAL_SERVER_ERROR,
            detail="Erro interno do servidor"
        )
```

## Boas Práticas

1. **Documentação**: Sempre documente endpoints com docstrings
2. **Validação**: Use Pydantic para validação de dados
3. **Tipos**: Sempre use type hints
4. **Separação**: Mantenha lógica de negócio fora dos endpoints
5. **Consistência**: Use os mesmos padrões em todo o projeto
6. **Testes**: Cada endpoint deve ter testes correspondentes
7. **Logging**: Implemente logging adequado
8. **Paginação**: Implemente paginação para listagens
9. **Versionamento**: Use versionamento de API
10. **Status Codes**: Use códigos HTTP apropriados
