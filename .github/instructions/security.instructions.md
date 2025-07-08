---
applyTo: '**'
---

# Segurança em APIs FastAPI

## Autenticação e Autorização

### JWT (JSON Web Token)
```python
from datetime import datetime, timedelta
from typing import Optional
from jose import JWTError, jwt
from passlib.context import CryptContext
from fastapi import HTTPException, status, Depends
from fastapi.security import HTTPBearer, HTTPAuthorizationCredentials

# Configurações de segurança
SECRET_KEY = "your-secret-key-here"  # Use variável de ambiente
ALGORITHM = "HS256"
ACCESS_TOKEN_EXPIRE_MINUTES = 30

pwd_context = CryptContext(schemes=["bcrypt"], deprecated="auto")
security = HTTPBearer()

class TokenData:
    def __init__(self, username: Optional[str] = None):
        self.username = username

def verify_password(plain_password: str, hashed_password: str) -> bool:
    """Verificar senha."""
    return pwd_context.verify(plain_password, hashed_password)

def get_password_hash(password: str) -> str:
    """Gerar hash da senha."""
    return pwd_context.hash(password)

def create_access_token(data: dict, expires_delta: Optional[timedelta] = None) -> str:
    """Criar token de acesso."""
    to_encode = data.copy()
    if expires_delta:
        expire = datetime.utcnow() + expires_delta
    else:
        expire = datetime.utcnow() + timedelta(minutes=15)
    
    to_encode.update({"exp": expire})
    encoded_jwt = jwt.encode(to_encode, SECRET_KEY, algorithm=ALGORITHM)
    return encoded_jwt

async def verify_token(credentials: HTTPAuthorizationCredentials = Depends(security)) -> TokenData:
    """Verificar token JWT."""
    credentials_exception = HTTPException(
        status_code=status.HTTP_401_UNAUTHORIZED,
        detail="Token inválido",
        headers={"WWW-Authenticate": "Bearer"},
    )
    
    try:
        payload = jwt.decode(credentials.credentials, SECRET_KEY, algorithms=[ALGORITHM])
        username: str = payload.get("sub")
        if username is None:
            raise credentials_exception
        token_data = TokenData(username=username)
    except JWTError:
        raise credentials_exception
    
    return token_data
```

### OAuth2 com Scopes
```python
from fastapi import Depends, HTTPException, status
from fastapi.security import OAuth2PasswordBearer, OAuth2PasswordRequestForm
from typing import List, Optional

oauth2_scheme = OAuth2PasswordBearer(
    tokenUrl="token",
    scopes={
        "read": "Leitura de dados",
        "write": "Escrita de dados",
        "admin": "Acesso administrativo"
    }
)

class User:
    def __init__(self, username: str, email: str, scopes: List[str] = None):
        self.username = username
        self.email = email
        self.scopes = scopes or []

async def get_current_user(token: str = Depends(oauth2_scheme)) -> User:
    """Obter usuário atual do token."""
    credentials_exception = HTTPException(
        status_code=status.HTTP_401_UNAUTHORIZED,
        detail="Token inválido",
        headers={"WWW-Authenticate": "Bearer"},
    )
    
    try:
        payload = jwt.decode(token, SECRET_KEY, algorithms=[ALGORITHM])
        username: str = payload.get("sub")
        scopes: List[str] = payload.get("scopes", [])
        
        if username is None:
            raise credentials_exception
            
        # Buscar usuário no banco de dados
        user = await get_user_by_username(username)
        if user is None:
            raise credentials_exception
            
        user.scopes = scopes
        return user
    except JWTError:
        raise credentials_exception

def require_scopes(required_scopes: List[str]):
    """Decorator para verificar permissões."""
    def decorator(current_user: User = Depends(get_current_user)):
        if not all(scope in current_user.scopes for scope in required_scopes):
            raise HTTPException(
                status_code=status.HTTP_403_FORBIDDEN,
                detail="Permissões insuficientes"
            )
        return current_user
    return decorator
```

## Validação de Entrada

### Sanitização de Dados
```python
from pydantic import BaseModel, validator, Field
from typing import Optional
import re

class UserCreate(BaseModel):
    email: str = Field(..., max_length=255)
    name: str = Field(..., min_length=2, max_length=100)
    password: str = Field(..., min_length=8, max_length=128)
    
    @validator('email')
    def validate_email(cls, v):
        # Validação de formato de email
        email_pattern = r'^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$'
        if not re.match(email_pattern, v):
            raise ValueError('Formato de email inválido')
        return v.lower().strip()
    
    @validator('name')
    def validate_name(cls, v):
        # Remover caracteres especiais
        v = re.sub(r'[<>\"\'&]', '', v)
        if len(v.strip()) < 2:
            raise ValueError('Nome deve ter pelo menos 2 caracteres')
        return v.strip()
    
    @validator('password')
    def validate_password(cls, v):
        # Validação de força da senha
        if len(v) < 8:
            raise ValueError('Senha deve ter pelo menos 8 caracteres')
        if not re.search(r'[A-Z]', v):
            raise ValueError('Senha deve conter pelo menos uma letra maiúscula')
        if not re.search(r'[a-z]', v):
            raise ValueError('Senha deve conter pelo menos uma letra minúscula')
        if not re.search(r'\d', v):
            raise ValueError('Senha deve conter pelo menos um número')
        return v

class QueryParams(BaseModel):
    page: int = Field(default=1, ge=1, le=1000)
    limit: int = Field(default=10, ge=1, le=100)
    search: Optional[str] = Field(default=None, max_length=100)
    
    @validator('search')
    def validate_search(cls, v):
        if v:
            # Sanitizar busca para prevenir SQL injection
            v = re.sub(r'[^\w\s-]', '', v)
            return v.strip()
        return v
```

## Proteção contra Ataques

### Rate Limiting
```python
from fastapi import HTTPException, Request
from slowapi import Limiter, _rate_limit_exceeded_handler
from slowapi.util import get_remote_address
from slowapi.errors import RateLimitExceeded
from slowapi.middleware import SlowAPIMiddleware

limiter = Limiter(key_func=get_remote_address)

# Middleware para rate limiting
app.state.limiter = limiter
app.add_exception_handler(RateLimitExceeded, _rate_limit_exceeded_handler)
app.add_middleware(SlowAPIMiddleware)

@app.post("/login")
@limiter.limit("5/minute")
async def login(request: Request, credentials: UserCredentials):
    """Endpoint de login com rate limiting."""
    # Lógica de autenticação
    pass

@app.get("/api/data")
@limiter.limit("100/minute")
async def get_data(request: Request):
    """Endpoint com rate limiting."""
    # Lógica para obter dados
    pass
```

### CORS (Cross-Origin Resource Sharing)
```python
from fastapi.middleware.cors import CORSMiddleware

app.add_middleware(
    CORSMiddleware,
    allow_origins=["https://exemplo.com"],  # Específico para produção
    allow_credentials=True,
    allow_methods=["GET", "POST", "PUT", "DELETE"],
    allow_headers=["*"],
)
```

### Middleware de Segurança
```python
from fastapi import Request, Response
from starlette.middleware.base import BaseHTTPMiddleware
import time

class SecurityMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request: Request, call_next):
        # Headers de segurança
        response = await call_next(request)
        
        # Prevenir clickjacking
        response.headers["X-Frame-Options"] = "DENY"
        
        # Prevenir MIME type sniffing
        response.headers["X-Content-Type-Options"] = "nosniff"
        
        # XSS Protection
        response.headers["X-XSS-Protection"] = "1; mode=block"
        
        # Política de segurança de conteúdo
        response.headers["Content-Security-Policy"] = "default-src 'self'"
        
        # HSTS (para HTTPS)
        response.headers["Strict-Transport-Security"] = "max-age=31536000; includeSubDomains"
        
        return response

app.add_middleware(SecurityMiddleware)
```

## Logging e Monitoramento

### Logging de Segurança
```python
import logging
from fastapi import Request
from datetime import datetime

# Configurar logger de segurança
security_logger = logging.getLogger("security")
security_logger.setLevel(logging.INFO)

handler = logging.FileHandler("security.log")
formatter = logging.Formatter(
    '%(asctime)s - %(name)s - %(levelname)s - %(message)s'
)
handler.setFormatter(formatter)
security_logger.addHandler(handler)

async def log_security_event(request: Request, event: str, details: dict = None):
    """Registrar evento de segurança."""
    log_data = {
        "timestamp": datetime.utcnow().isoformat(),
        "event": event,
        "ip": request.client.host,
        "user_agent": request.headers.get("user-agent"),
        "path": request.url.path,
        "method": request.method,
        "details": details or {}
    }
    
    security_logger.info(f"Security Event: {log_data}")

# Exemplo de uso
async def authenticate_user(request: Request, credentials: UserCredentials):
    """Autenticar usuário com logging."""
    try:
        user = await verify_credentials(credentials)
        await log_security_event(request, "LOGIN_SUCCESS", {"user": user.username})
        return user
    except Exception as e:
        await log_security_event(request, "LOGIN_FAILED", {"error": str(e)})
        raise HTTPException(status_code=401, detail="Credenciais inválidas")
```

## Configuração de Ambiente

### Variáveis de Ambiente
```python
from pydantic import BaseSettings
from typing import List

class Settings(BaseSettings):
    secret_key: str
    algorithm: str = "HS256"
    access_token_expire_minutes: int = 30
    database_url: str
    allowed_origins: List[str] = ["https://exemplo.com"]
    debug: bool = False
    
    class Config:
        env_file = ".env"
        env_file_encoding = "utf-8"

settings = Settings()
```

### Arquivo .env
```bash
SECRET_KEY=sua-chave-secreta-muito-forte-aqui
DATABASE_URL=postgresql://user:password@localhost/dbname
ALLOWED_ORIGINS=https://exemplo.com,https://app.exemplo.com
DEBUG=False
```

## Criptografia

### Criptografia de Dados Sensíveis
```python
from cryptography.fernet import Fernet
import base64

class DataEncryption:
    def __init__(self, key: str):
        self.key = key.encode()
        self.fernet = Fernet(base64.urlsafe_b64encode(self.key[:32]))
    
    def encrypt(self, data: str) -> str:
        """Criptografar dados."""
        return self.fernet.encrypt(data.encode()).decode()
    
    def decrypt(self, encrypted_data: str) -> str:
        """Descriptografar dados."""
        return self.fernet.decrypt(encrypted_data.encode()).decode()

# Uso
encryption = DataEncryption(settings.secret_key)
encrypted_data = encryption.encrypt("dados sensíveis")
decrypted_data = encryption.decrypt(encrypted_data)
```

## Checklist de Segurança

### Desenvolvimento
- [ ] Usar HTTPS em produção
- [ ] Implementar autenticação JWT
- [ ] Validar todas as entradas
- [ ] Sanitizar dados de entrada
- [ ] Implementar rate limiting
- [ ] Configurar CORS adequadamente
- [ ] Usar headers de segurança
- [ ] Implementar logging de segurança
- [ ] Criptografar dados sensíveis
- [ ] Usar variáveis de ambiente para secrets

### Produção
- [ ] Configurar firewall
- [ ] Monitorar logs de segurança
- [ ] Implementar backup seguro
- [ ] Configurar SSL/TLS
- [ ] Usar proxy reverso
- [ ] Implementar WAF
- [ ] Monitorar tentativas de ataque
- [ ] Manter dependências atualizadas
- [ ] Implementar 2FA para admin
- [ ] Realizar testes de penetração

### Boas Práticas
1. **Princípio do Menor Privilégio**: Conceder apenas permissões necessárias
2. **Defesa em Profundidade**: Múltiplas camadas de segurança
3. **Fail Secure**: Falhar de forma segura
4. **Auditoria**: Registrar todas as ações importantes
5. **Atualizações**: Manter sistema sempre atualizado
6. **Testes**: Testar regularmente a segurança
7. **Treinamento**: Educar equipe sobre segurança
8. **Incidente Response**: Ter plano de resposta a incidentes
