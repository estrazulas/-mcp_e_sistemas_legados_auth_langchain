# Using MCP with LangChain — Multi-Tools Agent + LangGraph + LangSmith

> **⚠️ Projeto com fins educacionais.** Desenvolvido durante as aulas da pós-graduação [Engenharia de Software com IA Aplicada (UNIP)](https://github.com/unipds-engenharia-de-ia-aplicada/engenharia-de-software-com-ia-aplicada). Destinado exclusivamente a testes e aprendizado — **não utilize em produção**.

Este projeto é uma evolução do repositório [mcp_e_sistemas_legados](https://github.com/estrazulas/mcp_e_sistemas_legados_auth_npm), substituindo o consumo direto do MCP pelo VS Code Copilot por uma camada de orquestração com **LangChain + LangGraph**, expondo o agente via **LangSmith Studio** para interação por chat.

---

## O que este projeto demonstra

- **MCP Server como LangChain Tool** — servidores MCP (`customers-mcp`, `filesystem`) são consumidos via `@langchain/mcp-adapters` e registrados como ferramentas no ecossistema LangChain
- **LangGraph Agent** — grafo de decisão com nós especializados (agente reativo com tools) que orquestra chamadas às ferramentas MCP
- **LangSmith Studio (Chat + Tracing)** — deploy local via `@langchain/langgraph-cli` com interface de chat e tracing completo das execuções
- **MultiServerMCPClient** — cliente unificado para conectar múltiplos servidores MCP simultaneamente (customers + filesystem)
- **OpenRouter + LangChain** — modelo LLM roteado pelo OpenRouter (provider routing) integrado ao agente LangChain

---

## Estrutura do projeto

```
├── 01-multiple-mcp-tools-z/        # LangGraph Agent — orquestra ferramentas MCP
│   ├── src/
│   │   ├── index.ts                # Entry point (Fastify + teste inline do /chat)
│   │   ├── server.ts               # Servidor Fastify com endpoint POST /chat
│   │   ├── config.ts               # Configuração do modelo LLM (OpenRouter)
│   │   ├── graph/
│   │   │   ├── factory.ts          # Factory do grafo (export para langgraph.json)
│   │   │   ├── graph.ts            # Definição do StateGraph (nós e arestas)
│   │   │   ├── state.ts            # Annotation e tipagem do estado do grafo
│   │   │   └── nodes/
│   │   │       └── agentNode.ts    # Nó agente: system prompt + LLM + tools
│   │   ├── services/
│   │   │   ├── mcpService.ts       # MultiServerMCPClient — conecta tools MCP
│   │   │   └── openRouterService.ts # Serviço LLM com createAgent e tools
│   │   ├── tools/
│   │   │   ├── customersTool.ts    # Config do MCP server de customers
│   │   │   └── fsTool.ts           # Config do MCP server de filesystem
│   │   └── prompts/v1/
│   │       └── agentNode.ts        # System prompt do agente
│   ├── data/
│   │   └── users.json              # Dados persistidos pelo filesystem MCP
│   ├── langgraph.json              # Config do LangGraph CLI (serve)
│   ├── .env                        # Variáveis de ambiente (LangSmith, OpenRouter, Service Token)
│   └── package.json                # Scripts: langgraph:serve, dev, start
│
└── nodejs-fastify-mongodb-crud-z/  # API REST (Fastify + MongoDB)
    ├── src/
    │   ├── index.js                # Servidor Fastify com rotas CRUD
    │   ├── auth.js                 # Autenticação JWT, service tokens e RBAC
    │   ├── config.js               # Configuração de banco e rate limit
    │   └── db.js                   # Conexão com MongoDB
    └── docker-compose.yml          # MongoDB para desenvolvimento local
```

---

## Comparação com o projeto 07 (API Security, Auth & Rate Limiting)

| Característica | Projeto 07 (MCP Direto) | Projeto 09 (MCP + LangChain) |
|---|---|---|
| **Consumo do MCP** | VS Code Copilot (stdio) | LangChain via `@langchain/mcp-adapters` |
| **Orquestração** | Copilot decide quais tools chamar | LangGraph StateGraph com nós especializados |
| **Interface de chat** | Copilot Chat no VS Code | LangSmith Studio (UI web) |
| **Tracing** | Limitado ao Copilot | LangSmith tracing completo (todas as tool calls) |
| **Deploy** | Config manual no `.vscode/mcp.json` | `npm run langgraph:serve` (LangGraph CLI) |
| **Múltiplos MCPs** | Configuração individual por server | `MultiServerMCPClient` unificado |
| **LLM** | Modelo do Copilot | OpenRouter (roteamento dinâmico de provider) |

---

## Como executar localmente

### Pré-requisitos

- Node.js v24+ (para o LangGraph Agent)
- Node.js >=20 (para a API REST)
- MongoDB (ou Docker)
- Conta no [LangSmith](https://smith.langchain.com) (API key)
- Conta no [OpenRouter](https://openrouter.ai) (API key)

### 1. Subir a API REST (MongoDB + Fastify)

```bash
cd nodejs-fastify-mongodb-crud-z

# Iniciar MongoDB via Docker
npm run infra:up:db

# Iniciar a API
npm start
```

A API estará disponível em `http://localhost:9999`.

### 2. Obter um Service Token

O MCP de customers (`@erickwendel/ew-customers-mcp`) autentica todas as chamadas à API REST com um token de serviço.

```bash
cd 01-multiple-mcp-tools-z
bash getServiceToken.sh
```

Isso chama `POST /v1/auth/service-token` e grava o token no arquivo `.env`.

### 3. Configurar o ambiente

Preencha o arquivo `01-multiple-mcp-tools-z/.env`:

```env
LANGSMITH_API_KEY=lsv2_pt_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
LANGCHAIN_TRACING_V2=true
LANGCHAIN_PROJECT=transforming-services-into-tools

OPENROUTER_API_KEY=sk-or-v1-xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
SERVICE_TOKEN=xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
```

> **Importante:** O `SERVICE_TOKEN` é obtido no passo 2. As demais chaves são das plataformas LangSmith e OpenRouter.

### 4. Rodar o LangGraph Agent (modo LangSmith Studio)

```bash
cd 01-multiple-mcp-tools-z
npm run langgraph:serve
```

Isso executa `npx @langchain/langgraph-cli dev`, que:

- Lê o arquivo `langgraph.json` para localizar o grafo (`./src/graph/factory.ts:graph`)
- Inicia um servidor local com a API do LangGraph
- Conecta ao LangSmith para tracing e chat
- Abre o **LangSmith Studio** no navegador (interface de chat)

---

## Interagindo via LangSmith Studio (Chat)

Com o `npm run langgraph:serve` rodando, acesse o [LangSmith Studio](https://smith.langchain.com/studio) e selecione o deployment local. No chat, você pode enviar perguntas em linguagem natural. O agente LangChain decide automaticamente quais ferramentas MCP invocar.

### Exemplo 1 — Listar todos os clientes

> **Prompt:** "Liste todos os clientes cadastrados"

**O que acontece internamente:**
1. O agente recebe a mensagem e identifica a intenção como `customer_operations`
2. O LLM decide invocar a tool MCP `customers-mcp` → `list_customers`
3. O `MultiServerMCPClient` encaminha a chamada via stdio para o processo `@erickwendel/ew-customers-mcp`
4. O MCP server chama a API REST (`GET /v1/customers`) com o `SERVICE_TOKEN`
5. O resultado retorna como resposta do chat no LangSmith Studio

**Resposta esperada:**
```
Clientes cadastrados:

1. João Silva — (11) 99999-0001
2. Maria Santos — (11) 99999-0002
3. Carlos Oliveira — (11) 99999-0003
```

O **tracing no LangSmith** mostra cada passo: `agent → tool:list_customers → api:GET /v1/customers → response`.

---

### Exemplo 2 — Adicionar um novo cliente

> **Prompt:** "Adicione o cliente Fernando Costa com telefone 11988887777"

**O que acontece internamente:**
1. O agente interpreta o prompt e decide usar a tool `create_customer`
2. A tool MCP é invocada com `{ name: "Fernando Costa", phone: "11988887777" }`
3. O MCP server chama `POST /v1/customers` na API REST
4. O cliente é persistido no MongoDB e retornado

**Resposta esperada:**
```
Cliente criado com sucesso:

Nome: Fernando Costa
Telefone: 11988887777
ID: 6a1f6c70ad367ff243def40e
```

**Verificação:** Pergunte em seguida "Liste todos os clientes novamente" e veja o novo cliente aparecer na listagem.

---

### Exemplo 3 — Remover um cliente por nome

> **Prompt:** "Remova o cliente chamado Maria Santos"

**O que acontece internamente:**
1. O agente primeiro chama `list_customers` para encontrar o cliente pelo nome
2. Identifica o `_id` correspondente à "Maria Santos"
3. Decide invocar `delete_customer` com o ID encontrado
4. O MCP server chama `DELETE /v1/customers/:id` (exige role `admin` no service token)
5. Confirma a exclusão

**Resposta esperada:**
```
Cliente removido com sucesso:

Nome: Maria Santos
ID: 6a1f6c70ad367ff243def40c
```

> **Nota sobre RBAC:** A operação de DELETE exige um service token com role `admin`. Certifique-se de que o token foi gerado com `adminSuperSecret` correto (padrão: `"AM I THE BOSS?"`).

---

### Exemplo 4 — Operação mista com persistência em arquivo

> **Prompt:** "Crie 3 clientes de teste, depois salve a lista completa de clientes no arquivo data/users.json e me mostre o conteúdo"

**O que acontece internamente:**
1. O agente chama `create_customer` 3 vezes (gera nomes/telefones de teste automaticamente)
2. Em seguida, chama `list_customers` para obter a lista atualizada
3. Decide usar a tool `filesystem` → `write_file` para salvar em `data/users.json`
4. Opcionalmente, chama `filesystem` → `read_file` para confirmar o conteúdo

Este exemplo demonstra o uso simultâneo de **dois servidores MCP** diferentes (customers + filesystem) orquestrados pelo mesmo agente LangChain.

---


## Rodando com o servidor Fastify local (alternativa ao LangSmith Studio)

Se preferir testar sem o LangSmith Studio, o projeto também expõe um endpoint HTTP tradicional:

```bash
cd 01-multiple-mcp-tools-z
npm start
```

Isso inicia o servidor Fastify em `http://localhost:3000` e faz uma chamada de teste:

```bash
curl -X POST http://localhost:3000/chat \
  -H "Content-Type: application/json" \
  -d '{"question": "Liste todos os clientes cadastrados"}'
```

---

## Testes

### API REST

```bash
cd nodejs-fastify-mongodb-crud-z
npm test
```

---

## Links

- [Repositório da pós-graduação (UNIP)](https://github.com/unipds-engenharia-de-ia-aplicada/engenharia-de-software-com-ia-aplicada)
- [Projeto predecessor — mcp_e_sistemas_legados_auth_npm](https://github.com/estrazulas/https://github.com/estrazulas/mcp_e_sistemas_legados_auth_npm)
- [Projeto 07 — API Security, Auth & Rate Limiting (MCP Direto)](../07-api-security-auth-rate-limiting-template)
- [Model Context Protocol (MCP)](https://modelcontextprotocol.io)
- [LangChain MCP Adapters](https://js.langchain.com/docs/integrations/tools/mcp)
- [LangGraph](https://langchain-ai.github.io/langgraphjs/)
- [LangSmith Studio](https://smith.langchain.com/studio)
- [LangGraph CLI](https://langchain-ai.github.io/langgraphjs/how-tos/langgraph-cli/)
- [OpenRouter](https://openrouter.ai)
- [Fastify](https://fastify.dev)
