# Guardian Agent — Porteiro Virtual Inteligente

> Sistema em produção · Código proprietário · Repositório público apenas para referência

Agente de IA conversacional via WhatsApp que atua como porteiro virtual para condomínios, desenvolvido sob medida para uma empresa de portaria remota. Opera com um único número central atendendo múltiplos condomínios simultaneamente (arquitetura multi-tenant).

**Meta de resolução automática: 85% das interações sem intervenção humana.**

---

## O que o sistema faz

- Moradores autorizam visitantes via WhatsApp em linguagem natural
- O agente interpreta a intenção, executa ações (pré-liberação, consulta de visitantes frequentes, abertura de ocorrências) via **function calling** com o modelo LLM
- Escala automaticamente para operador humano quando necessário, com fluxo de revisão D+1
- Síndicos têm acesso diferenciado com ferramentas específicas para gestão do condomínio
- Consome e integra com sistema legado de controle de acesso (Situator) via API

---

## Stack

| Camada | Tecnologia |
|--------|-----------|
| Runtime | Node.js + TypeScript |
| API | Express |
| LLM | Google Gemini 2.5 Flash via Vertex AI (function calling + Structured Outputs) |
| Banco de dados | PostgreSQL + Prisma ORM |
| Fila assíncrona | Redis + BullMQ |
| Mensageria | Meta Cloud API (WhatsApp Business) |
| Storage | Google Cloud Storage (anexos de ocorrências, signed URLs) |
| Deploy | Google Cloud (Cloud Run) |

---

## Arquitetura

Padrão **CSMR** — Controller → Service → Model → Repository — com injeção de dependência manual via factories. Estrutura modular em `src/modules/[contexto]/`.

```
src/
├── modules/
│   ├── agent/          # Pipeline LLM: GeminiService, ConversationService, ToolExecutorService
│   ├── auth/           # JWT, rate limiting, middlewares
│   ├── whatsapp/       # Webhook HMAC, worker BullMQ
│   ├── resident/       # CRUD moradores
│   ├── visitor/        # Pré-autorização de visitantes
│   ├── occurrence/     # Abertura de ocorrências + upload de anexos
│   ├── device/         # Dispositivos de acesso por morador
│   ├── syndic/         # Identificação e ferramentas do síndico
│   ├── metrics/        # Métricas operacionais com RBAC
│   └── ...             # +4 módulos
└── lib/
    ├── CircuitBreaker  # Circuit breakers para Gemini, Situator e Meta APIs
    ├── config          # Validação fail-fast de variáveis de ambiente
    └── logger          # Pino structured logging (LGPD-safe)
```

---

## Destaques técnicos

**Resiliência**
- Circuit breakers implementados do zero para as 3 APIs externas (Gemini, Meta, Situator)
- Retry com backoff exponencial nas chamadas à Meta API
- Rate limiting por telefone via sliding window no Redis
- Fallbacks no `ToolExecutorService` para degradação graciosa

**Segurança**
- Auditoria OWASP completa nos módulos de maior risco (Tier 1 + Tier 2)
- HMAC-SHA256 na validação do webhook do WhatsApp
- IDOR fix com validação de `tenantId` em todas as queries
- Login rate limiter + política de senha configurável (`BCRYPT_ROUNDS`)
- Threat modeling formal (TM-001, TM-002) com delimitadores contra LLM injection

**Conformidade (LGPD)**
- `MessageRetentionQueue` + worker para expiração automática de histórico de mensagens
- Campo `retainUntil` com TTL configurável via env (`LGPD_RETENTION_DAYS_MESSAGES`)
- LLM tracing sem PII

**Qualidade**
- Suite Jest completa sem dependências externas (ioredis-mock, fetch mocks)
- 34 requisitos de produção cobertos: CircuitBreaker, HMAC, rate limit, isolamento de tenant, smoke tests
- `InMemoryRepository` first — desenvolvimento desacoplado de banco antes de `PrismaRepository`

---

## O que aprendi construindo isso

- Projetar um pipeline LLM com function calling em produção: gerenciar histórico de contexto, token budget, Structured Outputs para respostas determinísticas
- Multi-tenancy com isolamento por `tenantId` em camada de repositório — qualquer query sem o campo é erro de compilação
- Circuit breaker como padrão essencial para APIs de terceiros (não só retry)
- LGPD aplicada a sistemas de conversação: o que pode e não pode ser retido
- Threat modeling antes de implementar, não depois

---

## Status

Em desenvolvimento desde fevereiro/março de 2026. Desenvolvimento contínuo — atualmente na Fase 9 (suporte completo a síndico + painel web para operadores).

---

*Código proprietário — este repositório existe apenas para referência de portfólio.*
