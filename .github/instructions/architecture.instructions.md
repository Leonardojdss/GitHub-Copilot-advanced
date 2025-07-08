---
applyTo: '**'
---

# Arquitetura Clean Code com FastAPI

## Estrutura de Pastas

```
project/
├── env/                        # ambiente virtual
├── app/
│   ├── __init__.py
│   ├── main.py                 # Ponto de entrada da aplicação
│   ├── core/
│   │   ├── __init__.py
│   │   ├── config.py          # Configurações da aplicação
│   │   ├── security.py        # Utilitários de segurança
│   │   └── database.py        # Configuração do banco de dados
│   ├── domain/
│   │   ├── __init__.py
│   │   ├── entities/          # Entidades de domínio
│   │   ├── repositories/      # Interfaces dos repositórios
│   │   └── services/          # Serviços de domínio
│   ├── infrastructure/
│   │   ├── __init__.py
│   │   ├── database/          # Implementação de repositórios
│   │   ├── external/          # Integrações externas
│   │   └── persistence/       # Modelos de persistência
│   ├── presentation/
│   │   ├── __init__.py
│   │   ├── api/
│   │   │   ├── __init__.py
│   │   │   ├── v1/            # Versionamento da API
│   │   │   └── dependencies.py # Dependências do FastAPI
│   │   └── schemas/           # Modelos Pydantic
│   └── tests/
│       ├── __init__.py
│       ├── unit/
│       ├── integration/
│       └── e2e/
```

* Sempre criar ambientes virtuais para isolar dependências

## Princípios SOLID Aplicados

### Single Responsibility Principle (SRP)
- Cada classe tem uma única responsabilidade
- Separação clara entre domínio, apresentação e infraestrutura
- Serviços específicos para cada funcionalidade

### Open/Closed Principle (OCP)
- Interfaces para extensibilidade
- Uso de dependências abstratas
- Facilita adição de novos recursos sem modificar código existente

### Liskov Substitution Principle (LSP)
- Implementações de repositórios intercambiáveis
- Herança adequada entre entidades
- Polimorfismo respeitado

### Interface Segregation Principle (ISP)
- Interfaces específicas e coesas
- Evita dependências desnecessárias
- Contratos bem definidos

### Dependency Inversion Principle (DIP)
- Dependências por abstrações
- Inversão de controle com FastAPI
- Facilita testes e manutenção

## Camadas da Arquitetura

### Domain Layer
- **Entities**: Objetos de negócio puros
- **Repositories**: Interfaces para acesso a dados
- **Services**: Lógica de negócio

### Infrastructure Layer
- **Database**: Implementação de repositórios
- **External**: APIs externas e integrações
- **Persistence**: Modelos de banco de dados

### Presentation Layer
- **API Routes**: Endpoints REST
- **Schemas**: Validação e serialização
- **Dependencies**: Injeção de dependências

## Exemplo de Implementação

```python
# domain/entities/user.py
from dataclasses import dataclass
from typing import Optional
from datetime import datetime

@dataclass
class User:
    id: Optional[int] = None
    email: str = ""
    name: str = ""
    created_at: Optional[datetime] = None
    
    def is_valid(self) -> bool:
        return bool(self.email and self.name)

# domain/repositories/user_repository.py
from abc import ABC, abstractmethod
from typing import List, Optional
from domain.entities.user import User

class UserRepository(ABC):
    @abstractmethod
    async def create(self, user: User) -> User:
        pass
    
    @abstractmethod
    async def get_by_id(self, user_id: int) -> Optional[User]:
        pass
    
    @abstractmethod
    async def get_by_email(self, email: str) -> Optional[User]:
        pass
    
    @abstractmethod
    async def list_all(self) -> List[User]:
        pass

# domain/services/user_service.py
from domain.entities.user import User
from domain.repositories.user_repository import UserRepository

class UserService:
    def __init__(self, user_repository: UserRepository):
        self._user_repository = user_repository
    
    async def create_user(self, user: User) -> User:
        if not user.is_valid():
            raise ValueError("Dados de usuário inválidos")
        
        existing_user = await self._user_repository.get_by_email(user.email)
        if existing_user:
            raise ValueError("Email já está em uso")
        
        return await self._user_repository.create(user)
```

## Benefícios

1. **Manutenibilidade**: Código organizado e fácil de modificar
2. **Testabilidade**: Isolamento de dependências facilita testes
3. **Escalabilidade**: Estrutura permite crescimento ordenado
4. **Flexibilidade**: Mudanças em uma camada não afetam outras
5. **Legibilidade**: Código expressivo e bem estruturado
