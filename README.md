# 🛰️ Cyber + Space Operations Agent

> Laboratório real de um **Agent 365** que roda no **Microsoft Teams**, consome **APIs públicas** (NVD CVE + NASA APOD) através de um **MCP Server próprio em Python**, é orquestrado por um **backend FastAPI + Azure AI Foundry**, e é **governado/registrado/observado** pelo **Microsoft Agent 365**.

\![status](https://img.shields.io/badge/status-lab-blue)
\![python](https://img.shields.io/badge/python-3.11%2B-green)
\![mcp](https://img.shields.io/badge/MCP-v1.x-orange)
\![license](https://img.shields.io/badge/license-MIT-lightgrey)

---

## ⚠️ Separação de planos (premissa do projeto)

Este repositório é o **runtime** do agente. O **Microsoft Agent 365 NÃO executa o agente** — ele atua como **registry, control plane, identity plane e observabilidade**.

> Validado em fonte oficial: blog *"Microsoft Agent 365: the control plane for AI agents"* (18/11/2025) e [Microsoft Learn](https://learn.microsoft.com/microsoft-agent-365/).

| Plano | O que é | Quem faz aqui |
|-------|---------|---------------|
| **Runtime** | Onde o agente realmente executa | Backend FastAPI + MCP Server + Azure AI Foundry |
| **Registry** | Catálogo central de agentes | Microsoft Agent 365 — *agent registry* |
| **Control plane** | Governança e ciclo de vida do fleet | Microsoft Agent 365 |
| **Identity plane** | Identidade do agente | Microsoft Entra Agent ID |

---

## 🏗️ Arquitetura

```text
Usuário (Microsoft Teams)
   │
   ▼
Teams App / Bot  ───────────────┐  (canal de conversa)
   │                            │
   ▼                            │
Backend Python (FastAPI)        │  ← RUNTIME (você hospeda: Container Apps / App Service)
   │   orquestração + auth      │
   ▼                            │
Azure AI Foundry (LLM)          │  ← raciocínio / linguagem
   │   "tool calling"           │
   ▼                            │
MCP Server próprio (Python)     │  ← 5 ferramentas padronizadas (MCP)
   │                            │
   ├──► NVD CVE API 2.0 (NIST)  │  ← vulnerabilidades
   └──► NASA APOD API           │  ← imagem astronômica do dia
   │
   ▼
Resposta ao usuário no Teams
   ┊
   └┄┄► Agent 365 → REGISTRY · IDENTITY (Entra Agent ID) · GOVERNANCE · OBSERVABILITY
        (control plane — NÃO é onde o agente executa)
```

---

## 🧰 Ferramentas MCP

| Ferramenta | Descrição | Fonte de dados |
|-----------|-----------|----------------|
| `search_cve` | Busca CVEs por palavra-chave e severidade | NVD CVE API 2.0 |
| `get_cve_details` | Detalhe de uma CVE por ID | NVD CVE API 2.0 |
| `get_nasa_apod` | Imagem astronômica do dia | NASA APOD |
| `summarize_risk` | Agrega risco a partir das CVEs (determinístico) | NVD + lógica local |
| `generate_executive_briefing` | Briefing executivo factual | NVD + lógica local |

> **Decisão de design:** as ferramentas MCP são **determinísticas** e **não chamam o LLM**. Elas devolvem *fato*; o LLM (Foundry) só produz *narrativa* e é instruído a **nunca inventar CVE ou score**.

---

## 📁 Estrutura do projeto

```text
cyber-space-agent/
├── app/
│   ├── [config.py](coworker-file://469112b2-7a21-515f-8df3-6d58159ab451/config.py)              # configuração via pydantic-settings (sem segredos no código)
│   ├── clients/               # NVD e NASA (httpx async)
│   ├── services/[risk.py](coworker-file://37fe603e-b973-5889-9f08-bc9e167d6938/risk.py)       # risco e briefing executivo (determinístico)
│   ├── mcp_server/[server.py](coworker-file://8fb9473d-b2c9-5010-9885-27f27fce08f2/server.py)   # MCP Server com 5 tools (FastMCP)
│   └── backend/               # FastAPI + integração Azure AI Foundry
├── observability/[otel.py](coworker-file://8b8d82d3-1623-55fc-a337-41ca7a8803b6/otel.py)      # OpenTelemetry (traces/metrics/logs)
├── teams/                     # manifesto do Teams App/Bot
├── tests/                     # testes offline (respx)
├── docs/                      # PLANOS · GUIA_AULA · ROTEIRO_DEMO · FAQ
├── [requirements.txt](coworker-file://f821006d-8216-5929-9397-b445b7e73fbc/requirements.txt)
└── .env.example
```

---

## ✅ Pré-requisitos

- Windows 11 + [VS Code](https://code.visualstudio.com/)
- [Python 3.11+](https://www.python.org/downloads/)
- Chave gratuita NASA: <https://api.nasa.gov/> (ou `DEMO_KEY` para testes)
- (Opcional) Chave NVD: <https://nvd.nist.gov/developers/request-an-api-key>
- Recurso **Azure AI Foundry** com um modelo implantado
- (Para Teams/Agent 365) Tenant Microsoft 365 com permissão de admin

---

## 🚀 Quickstart (local, sem Teams)

```powershell
# 1. Ambiente virtual
python -m venv .venv
.\.venv\Scripts\Activate.ps1

# 2. Dependências
pip install -r [requirements.txt](coworker-file://f821006d-8216-5929-9397-b445b7e73fbc/requirements.txt)

# 3. Segredos (NUNCA commite o .env real)
copy .env.example .env
#    edite .env e preencha NASA_API_KEY etc.

# 4. Testes (offline, com mocks)
pytest -q

# 5a. Subir o MCP Server (porta 8000)
python -m app.mcp_server.server
#     em outro terminal:  npx -y @modelcontextprotocol/inspector
#     conectar em http://localhost:8000/mcp

# 5b. OU subir o backend FastAPI (porta 8080)
uvicorn app.backend.main:app --reload --port 8080
#     teste:  http://localhost:8080/docs
```

---

## 🎬 Demonstrações

| # | Pergunta no Teams |
|---|-------------------|
| 1 | *"Quais CVEs críticas recentes existem sobre OpenSSL e qual o risco executivo?"* |
| 2 | *"Gere um briefing técnico de vulnerabilidades para o CISO."* |
| 3 | *"Mostre a imagem astronômica do dia da NASA e gere uma analogia com risco cibernético."* |
| 4 | *"Quais agentes estão registrados e qual runtime cada um usa?"* (vem do **Agent 365 Registry**, não das APIs públicas) |

Roteiro completo com comandos `curl` em [`docs/ROTEIRO_DEMO.md`](docs/ROTEIRO_DEMO.md).

---

## 🔌 Bedrock, Vertex e on-prem

- **Amazon Bedrock / Google Vertex AI** → via **Registry Sync** do Agent 365 (**preview**): o agente continua executando na nuvem de origem; passa a ser **catalogado** no registry. [Doc oficial](https://learn.microsoft.com/microsoft-agent-365/admin/agent-registry).
- **n8n / LangChain on-prem** → padrão recomendado: **Entra Agent ID** (identidade) + **SDK/API** (registro) + **sidecar OpenTelemetry** (observabilidade).

> ⚠️ Registry Sync é **preview**. Para agentes on-prem genéricos, confirme a disponibilidade no Learn no momento da implementação.

---

## 🔒 Segurança

- **Nenhum segredo no código** — tudo via variáveis de ambiente (`.env`), validadas por `pydantic-settings`.
- `.env` está no `.gitignore`. Use sempre `.env.example` como template.
- Em produção: **Azure Key Vault + Managed Identity** (sem chaves em arquivo).
- Timeouts e tratamento de erro em todas as chamadas externas.
- Valide o **JWT do Bot Framework** no endpoint `/api/messages`.

---

## 📚 Fontes oficiais

- NVD CVE API 2.0 — NIST: <https://nvd.nist.gov/developers/vulnerabilities>
- NASA APOD — <https://api.nasa.gov/>
- MCP Python SDK — <https://github.com/modelcontextprotocol/python-sdk>
- Microsoft Agent 365 — <https://learn.microsoft.com/microsoft-agent-365/>
- Azure AI Foundry — <https://learn.microsoft.com/azure/ai-foundry/>

---

## 📄 Licença

MIT. Veja [`LICENSE`](LICENSE).
