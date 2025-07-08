# GitHub Copilot: Demonstração de Funcionalidades Avançadas

Este repositório demonstra como utilizar as funcionalidades avançadas do GitHub Copilot, incluindo **chatmodes**, **prompt** e **instructions** para criar um ambiente de desenvolvimento mais eficiente e personalizado.

## 🚀 Funcionalidades Demonstradas

### Estrutura de Exemplo
```
.github/
├── instructions/
│   ├── architecture.instructions.md    # Arquitetura Clean Code com FastAPI
│   ├── code_style.instructions.md      # Padrões de código e convenções
│   ├── security.instructions.md        # Práticas de segurança e autenticação
│   ├── database.instructions.md        # Integração com banco de dados
│   ├── testing.instructions.md         # Estratégias de teste
│   └── deployment.instructions.md      # Deploy e DevOps
├── chatmodes/
│   └── dev-back.chatmode.md            # Chatmode especializado em backend
└── prompts/
    ├── criar-testes.prompt.md          # Prompt para criação de testes
    └── desenvolver.prompt.md           # Prompt para desenvolvimento de APIs
```

### 1. Instructions (Instruções)
As instruções permitem configurar comportamentos específicos do Copilot para seu projeto, definindo padrões de código, arquitetura e boas práticas.

#### Exemplo de Uso
- **Arquitetura**: Define estrutura Clean Code com FastAPI e princípios SOLID
- **Estilo de Código**: Estabelece padrões de nomenclatura e formatação
- **Segurança**: Configura práticas de segurança, JWT, OAuth2, rate limiting
- **Database**: Configuração SQLAlchemy, repositórios, modelos
- **Testing**: Estratégias de teste unitário, integração e e2e
- **Deployment**: Docker, Kubernetes, CI/CD, monitoramento

As instruções de prompt permitem personalizar como o Copilot interpreta e responde às suas solicitações, aplicando contexto específico ao seu projeto.

#### Configuração
```markdown
---
applyTo: '**' 
---

# Suas instruções personalizadas aqui
```

- **obs.** applyTo defini para quais pastas essas instruções devem ser levadas em consideração.

#### Benefícios
- ✅ Contexto específico do projeto
- ✅ Padrões de código consistentes
- ✅ Melhores sugestões de código
- ✅ Redução de erros comuns

### 2. Prompts
Os prompts permitem criar tarefas específicas, customo usar desta forma.

#### Configuração
```markdown
---
mode: agent
---

# Seu prompt personalizado aqui
```

#### Exemplo de Prompts
- **`/desenvolver`**: Workflow completo para desenvolvimento de APIs REST
- **`/criar-testes`**: Geração automatizada de testes com alta cobertura
- **Mode: agent**: Executa tarefas complexas de forma autônoma

#### Exemplos de Uso
- **Desenvolvimento**: Analisa requisitos → cria arquitetura → implementa código
- **Testes**: Analisa codebase → identifica cenários → gera testes completos
- **Refatoração**: Avalia código → sugere melhorias → implementa mudanças

#### Benefícios
- ✅ Automação de tarefas complexas
- ✅ Workflows estruturados e consistentes
- ✅ Redução de tempo em tarefas repetitivas
- ✅ Qualidade garantida por padrões definidos

### 3. Chatmodes
Os chatmodes são configurações predefinidas que permitem adaptar o comportamento de chat de IA na para tarefas específicas, como fazer perguntas, fazer edições de código ou executar tarefas de codificação autônomas, com ele você definir até mesmo quais ferramentas e MCPs ele pode utilizar.

## 🎯 Exemplos Práticos

### Criando um Endpoint
Quando você solicita ao Copilot (especialmente usando `@dev-back`) para criar um novo endpoint utilizando `/desenvolver`, ele automaticamente:
- Segue a arquitetura Clean Code definida
- Aplica os padrões de código estabelecidos
- Implementa as práticas de segurança configuradas (JWT, validação, rate limiting)
- Integra com banco de dados usando SQLAlchemy
- Mantém consistência com o resto do projeto

### Criando testes
Quando a criação do endpoint ser concluido você pode pedir para `/criar-testes`, ele vai gerar testes unitários e de integração com alta cobertura

### Documentação Oficial
- [Chat Modes - VS Code](https://code.visualstudio.com/docs/copilot/chat/chat-modes)
- [Custom Instructions - GitHub Docs](https://docs.github.com/en/copilot/how-tos/custom-instructions/adding-repository-custom-instructions-for-github-copilot?tool=vscode)

**Doc criada usando GitHub Copilot**