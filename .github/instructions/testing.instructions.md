∫

# Estratégias de Teste para FastAPI

## Estrutura de Testes

### Organização
```
tests/
├── __init__.py
├── conftest.py              # Configurações e fixtures globais
├── unit/                    # Testes unitários
│   ├── __init__.py
│   ├── domain/
│   │   ├── test_entities.py
│   │   └── test_services.py
│   └── infrastructure/
│       └── test_repositories.py
├── integration/             # Testes de integração
│   ├── __init__.py
│   ├── test_database.py
│   └── test_external_apis.py
└── e2e/                     # Testes end-to-end
    ├── __init__.py
    └── test_api_endpoints.py
```

## Configuração Base (conftest.py)

```python
import pytest
import asyncio
from typing import AsyncGenerator
from fastapi.testclient import TestClient
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker
from httpx import AsyncClient

from app.main import app
from app.core.database import get_db, Base
from app.core.config import settings

# Configuração do banco de teste
SQLALCHEMY_DATABASE_URL = "sqlite:///./test.db"
engine = create_engine(
    SQLALCHEMY_DATABASE_URL, 
    connect_args={"check_same_thread": False}
)
TestingSessionLocal = sessionmaker(autocommit=False, autoflush=False, bind=engine)

def override_get_db():
    """Override da dependência do banco de dados."""
    try:
        db = TestingSessionLocal()
        yield db
    finally:
        db.close()

app.dependency_overrides[get_db] = override_get_db

@pytest.fixture
def client():
    """Cliente de teste síncrono."""
    return TestClient(app)

@pytest.fixture
async def async_client() -> AsyncGenerator[AsyncClient, None]:
    """Cliente de teste assíncrono."""
    async with AsyncClient(app=app, base_url="http://test") as ac:
        yield ac

@pytest.fixture
def db_session():
    """Sessão de banco de dados para testes."""
    Base.metadata.create_all(bind=engine)
    db = TestingSessionLocal()
    try:
        yield db
    finally:
        db.close()
        Base.metadata.drop_all(bind=engine)

@pytest.fixture
def sample_user_data():
    """Dados de usuário para testes."""
    return {
        "email": "test@example.com",
        "name": "Test User",
        "password": "password123"
    }
```

## Testes Unitários

### Testando Entidades
```python
# tests/unit/domain/test_entities.py
import pytest
from datetime import datetime
from app.domain.entities.user import User

class TestUser:
    def test_user_creation(self):
        """Testar criação de usuário."""
        user = User(
            email="test@example.com",
            name="Test User",
            created_at=datetime.now()
        )
        assert user.email == "test@example.com"
        assert user.name == "Test User"
        assert user.created_at is not None

    def test_user_validation(self):
        """Testar validação de usuário."""
        # Usuário válido
        valid_user = User(email="test@example.com", name="Test User")
        assert valid_user.is_valid() is True
        
        # Usuário inválido - sem email
        invalid_user = User(email="", name="Test User")
        assert invalid_user.is_valid() is False
        
        # Usuário inválido - sem nome
        invalid_user = User(email="test@example.com", name="")
        assert invalid_user.is_valid() is False
```

### Testando Serviços
```python
# tests/unit/domain/test_services.py
import pytest
from unittest.mock import Mock, AsyncMock
from app.domain.entities.user import User
from app.domain.services.user_service import UserService
from app.domain.repositories.user_repository import UserRepository

class TestUserService:
    @pytest.fixture
    def mock_user_repository(self):
        """Mock do repositório de usuário."""
        return Mock(spec=UserRepository)
    
    @pytest.fixture
    def user_service(self, mock_user_repository):
        """Serviço de usuário com repositório mockado."""
        return UserService(mock_user_repository)
    
    async def test_create_user_success(self, user_service, mock_user_repository):
        """Testar criação de usuário com sucesso."""
        # Arrange
        user_data = User(email="test@example.com", name="Test User")
        created_user = User(id=1, email="test@example.com", name="Test User")
        
        mock_user_repository.get_by_email = AsyncMock(return_value=None)
        mock_user_repository.create = AsyncMock(return_value=created_user)
        
        # Act
        result = await user_service.create_user(user_data)
        
        # Assert
        assert result.id == 1
        assert result.email == "test@example.com"
        mock_user_repository.get_by_email.assert_called_once_with("test@example.com")
        mock_user_repository.create.assert_called_once_with(user_data)
    
    async def test_create_user_email_already_exists(self, user_service, mock_user_repository):
        """Testar criação de usuário com email já existente."""
        # Arrange
        user_data = User(email="test@example.com", name="Test User")
        existing_user = User(id=1, email="test@example.com", name="Existing User")
        
        mock_user_repository.get_by_email = AsyncMock(return_value=existing_user)
        
        # Act & Assert
        with pytest.raises(ValueError, match="Email já está em uso"):
            await user_service.create_user(user_data)
    
    async def test_create_user_invalid_data(self, user_service, mock_user_repository):
        """Testar criação de usuário com dados inválidos."""
        # Arrange
        invalid_user = User(email="", name="")
        
        # Act & Assert
        with pytest.raises(ValueError, match="Dados de usuário inválidos"):
            await user_service.create_user(invalid_user)
```

## Testes de Integração

### Testando Repositórios
```python
# tests/integration/test_repositories.py
import pytest
from app.domain.entities.user import User
from app.infrastructure.database.user_repository_impl import UserRepositoryImpl

class TestUserRepository:
    async def test_create_and_get_user(self, db_session):
        """Testar criação e busca de usuário."""
        # Arrange
        repository = UserRepositoryImpl(db_session)
        user_data = User(email="test@example.com", name="Test User")
        
        # Act
        created_user = await repository.create(user_data)
        retrieved_user = await repository.get_by_id(created_user.id)
        
        # Assert
        assert created_user.id is not None
        assert created_user.email == "test@example.com"
        assert retrieved_user is not None
        assert retrieved_user.email == "test@example.com"
    
    async def test_get_user_by_email(self, db_session):
        """Testar busca de usuário por email."""
        # Arrange
        repository = UserRepositoryImpl(db_session)
        user_data = User(email="test@example.com", name="Test User")
        
        # Act
        await repository.create(user_data)
        found_user = await repository.get_by_email("test@example.com")
        not_found_user = await repository.get_by_email("notfound@example.com")
        
        # Assert
        assert found_user is not None
        assert found_user.email == "test@example.com"
        assert not_found_user is None
```

## Testes End-to-End

### Testando Endpoints
```python
# tests/e2e/test_api_endpoints.py
import pytest
from fastapi.testclient import TestClient
from httpx import AsyncClient

class TestUserEndpoints:
    async def test_create_user_endpoint(self, async_client: AsyncClient, sample_user_data):
        """Testar endpoint de criação de usuário."""
        response = await async_client.post("/users/", json=sample_user_data)
        
        assert response.status_code == 201
        data = response.json()
        assert data["email"] == sample_user_data["email"]
        assert data["name"] == sample_user_data["name"]
        assert "id" in data
        assert "created_at" in data
    
    async def test_get_user_endpoint(self, async_client: AsyncClient, sample_user_data):
        """Testar endpoint de busca de usuário."""
        # Criar usuário
        create_response = await async_client.post("/users/", json=sample_user_data)
        user_id = create_response.json()["id"]
        
        # Buscar usuário
        get_response = await async_client.get(f"/users/{user_id}")
        
        assert get_response.status_code == 200
        data = get_response.json()
        assert data["id"] == user_id
        assert data["email"] == sample_user_data["email"]
    
    async def test_get_nonexistent_user(self, async_client: AsyncClient):
        """Testar busca de usuário inexistente."""
        response = await async_client.get("/users/999")
        
        assert response.status_code == 404
        assert "não encontrado" in response.json()["detail"]
    
    async def test_create_user_invalid_data(self, async_client: AsyncClient):
        """Testar criação de usuário com dados inválidos."""
        invalid_data = {
            "email": "invalid-email",
            "name": "",
            "password": "123"
        }
        
        response = await async_client.post("/users/", json=invalid_data)
        
        assert response.status_code == 422
        assert "detail" in response.json()
    
    async def test_list_users_endpoint(self, async_client: AsyncClient, sample_user_data):
        """Testar endpoint de listagem de usuários."""
        # Criar alguns usuários
        for i in range(3):
            user_data = sample_user_data.copy()
            user_data["email"] = f"user{i}@example.com"
            await async_client.post("/users/", json=user_data)
        
        # Listar usuários
        response = await async_client.get("/users/")
        
        assert response.status_code == 200
        data = response.json()
        assert len(data) == 3
        assert all("id" in user for user in data)
```

## Testes de Performance

```python
# tests/performance/test_load.py
import pytest
import asyncio
from httpx import AsyncClient

class TestPerformance:
    async def test_concurrent_user_creation(self, async_client: AsyncClient):
        """Testar criação concorrente de usuários."""
        async def create_user(index):
            user_data = {
                "email": f"user{index}@example.com",
                "name": f"User {index}",
                "password": "password123"
            }
            response = await async_client.post("/users/", json=user_data)
            return response.status_code
        
        # Criar 10 usuários concorrentemente
        tasks = [create_user(i) for i in range(10)]
        results = await asyncio.gather(*tasks)
        
        # Verificar que todos foram criados com sucesso
        assert all(status == 201 for status in results)
```

## Comandos de Teste

```bash
# Executar todos os testes
pytest

# Executar testes com cobertura
pytest --cov=app --cov-report=html

# Executar apenas testes unitários
pytest tests/unit/

# Executar apenas testes de integração
pytest tests/integration/

# Executar apenas testes e2e
pytest tests/e2e/

# Executar testes em paralelo
pytest -n auto

# Executar testes com output detalhado
pytest -v -s

# Executar teste específico
pytest tests/unit/domain/test_services.py::TestUserService::test_create_user_success
```

## Boas Práticas

1. **AAA Pattern**: Arrange, Act, Assert
2. **Mocks**: Use mocks para dependências externas
3. **Fixtures**: Reutilize configurações com fixtures
4. **Isolamento**: Cada teste deve ser independente
5. **Nomenclatura**: Nomes descritivos para testes
6. **Cobertura**: Mantenha cobertura de testes alta
7. **Performance**: Inclua testes de performance
8. **Dados**: Use dados de teste consistentes
9. **Cleanup**: Limpe dados após testes
10. **CI/CD**: Integre testes no pipeline
