---
mode: agent
---

# Criação de Testes Unitários e de Integração

## Objetivo

Você deve analisar completamente o repositório de código atual e criar uma suíte abrangente de testes unitários e de integração. Esta tarefa é fundamental para garantir a qualidade, confiabilidade e manutenibilidade do código.

## Instruções Detalhadas

### 1. Análise do Repositório

Antes de criar os testes, você deve:

- **Explorar a estrutura do projeto**: Identificar todos os arquivos de código, dependências, configurações e documentação
- **Entender a arquitetura**: Compreender como os componentes se relacionam e interagem
- **Identificar funcionalidades**: Mapear todas as funcionalidades implementadas, incluindo:
  - Funções e métodos públicos
  - Classes e seus comportamentos
  - APIs e endpoints
  - Fluxos de dados
  - Casos de uso principais

### 2. Estratégia de Testes

#### Testes Unitários
Criar testes que validem:
- **Funções individuais**: Testar cada função isoladamente
- **Métodos de classe**: Verificar comportamentos específicos de cada método
- **Casos extremos**: Testar valores limites, entradas inválidas e cenários de erro
- **Mocks e stubs**: Isolar dependências externas para testes focados

#### Testes de Integração
Desenvolver testes que verifiquem:
- **Integração entre componentes**: Como diferentes módulos trabalham juntos
- **APIs e endpoints**: Funcionamento correto das interfaces
- **Fluxos de dados**: Processamento completo de dados através do sistema
- **Integrações externas**: Conexões com bancos de dados, APIs externas, etc.

### 3. Cobertura de Testes

- **Cobertura de código**: Buscar pelo menos 80% de cobertura
- **Cobertura de funcionalidades**: Todos os casos de uso principais devem ser testados
- **Cenários de erro**: Incluir testes para tratamento de exceções e erros
- **Performance**: Quando relevante, incluir testes de performance

### 4. Estrutura e Organização

- **Nomenclatura clara**: Nomes descritivos para testes e suítes
- **Organização lógica**: Agrupar testes por funcionalidade ou módulo
- **Documentação**: Comentários explicando testes complexos
- **Configuração**: Setup e teardown adequados para cada teste

### 5. Ferramentas e Frameworks

- Utilizar as ferramentas de teste mais apropriadas para a linguagem/framework do projeto
- Configurar ambiente de testes adequado
- Implementar CI/CD para execução automática dos testes

### 6. Considerações Especiais

- **Dados de teste**: Criar dados de teste representativos mas não sensíveis
- **Ambientes**: Considerar diferentes ambientes (desenvolvimento, staging, produção)
- **Manutenibilidade**: Testes devem ser fáceis de manter e atualizar
- **Documentação**: Documentar estratégia de testes e como executá-los

## Resultado Esperado

Ao final, o repositório deve conter:
- Suíte completa de testes unitários e de integração
- Configuração adequada para execução dos testes
- Documentação sobre como executar e manter os testes
- Relatórios de cobertura de código

## Instruções de Execução

1. Analise completamente o código atual
2. Identifique gaps de teste existentes
3. Crie plano de testes abrangente
4. Implemente testes seguindo boas práticas
5. Configure ambiente de execução
6. Valide cobertura e qualidade dos testes
7. Documente processo e resultados