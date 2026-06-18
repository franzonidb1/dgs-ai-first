# Bounded Contexts — NovaTech Assistant

> **Localização no repo:** `docs/onboarding.md` — seção 1  
> **Público:** todo o time + agentes de IA (referenciado pelo AGENTS.md)  
> **Base:** Anexo A (documentação NovaTech) + discovery Cenário 1 + estrutura do repo (Anexo D)

---

## Por que este documento existe

Agentes de IA sem domínio claro geram outputs genéricos. Este documento define as fronteiras dentro das quais cada parte do sistema opera, quais responsabilidades pertencem a cada contexto, e o que está fora do escopo do assistente. Todo membro do time e todo agente deve ler este documento antes de escrever código, specs ou prompts para o projeto.

---

## Os quatro bounded contexts

### Contexto 1 — Ingestão de Documentos

**Diretório:** `src/pipeline/`  
**Specs:** `specs/pipeline-ingestao/`  
**Arquivos-chave:** `chunker.ts`, `embedder.ts`, `extractor.ts`, `indexer.ts`

**Missão:** transformar documentos brutos (PDFs, Word, páginas Confluence) em chunks indexados e pesquisáveis no Azure AI Search. Este contexto não sabe nada sobre perguntas de atendentes ou respostas do assistente — ele produz e mantém o corpus.

**Pertence a este contexto:**
- Extração de texto de documentos (extractor)
- Divisão em chunks com metadados de fonte (chunker) — atenção especial a tabelas, que quebraram no protótipo open-source (ADR-0004)
- Geração de embeddings vetoriais (embedder)
- Indexação no Azure AI Search (indexer)
- Detecção de conflito documental e marcação de `doc_status`
- Gestão do ciclo de vida dos documentos (vigente / histórico / pendente / vigente_transitório)
- Manutenção da lista de cobertura bloqueada (temas sem documento válido)

**Não pertence a este contexto:**
- Lógica de resposta ao atendente
- Regras de escalada
- Construção de prompt
- Interface visual

**Entidades centrais:**

| Entidade | Definição |
|---|---|
| `Document` | Documento bruto com metadados de origem (nome, código, versão, área responsável, data) |
| `Chunk` | Trecho indexável extraído de um `Document`, com metadados de rastreabilidade e vetor de embedding |
| `DocumentStatus` | Estado de vigência: `vigente` \| `histórico` \| `pendente_classificação` \| `vigente_transitório` |
| `ConflictRecord` | Registro de par de documentos que cobrem o mesmo tema com valores ou regras divergentes |
| `BlockedTopic` | Tema que não deve ser respondido pelo assistente enquanto o documento regulador está ausente ou em revisão |

---

### Contexto 2 — Consulta e Resposta *(módulo principal)*

**Diretório:** `src/functions/query/` + `src/services/`  
**Specs:** `specs/query-endpoint/`  
**Arquivos-chave:** `handler.ts`, `search.ts`, `prompt-builder.ts`, `completion.ts`, `response-validator.ts`

**Missão:** receber a pergunta do atendente, buscar os chunks mais relevantes, construir o prompt respeitando o context budget, chamar o modelo e retornar resposta com rastreabilidade de fonte. É o coração do assistente — o contexto que entrega valor ao atendente.

**Pertence a este contexto:**
- Validação e normalização da pergunta recebida (Zod)
- Busca por similaridade no índice (search service) com threshold de confiança
- Verificação da lista de tópicos bloqueados antes de usar os chunks recuperados
- Detecção de conflito nos chunks recuperados (dois chunks com valores divergentes para o mesmo campo)
- Construção do prompt com context budget: ~4K tokens system prompt + ~8K chunks (5 chunks × ~1.500 tokens) + pergunta + histórico de 3 turnos (ADR-0002)
- Chamada ao Azure OpenAI GPT-4o (ADR-0001)
- Validação da resposta gerada contra os guardrails
- Construção da resposta estruturada com citação de fonte (nome, versão, seção, data)
- Determinação do canal de escalada quando não há resposta segura

**Não pertence a este contexto:**
- Indexação de documentos (Contexto 1)
- Renderização de cards no Teams (Contexto 3)
- Persistência de feedback (Contexto 4)

**Entidades centrais:**

| Entidade | Definição |
|---|---|
| `Query` | Pergunta do atendente com contexto da sessão (histórico de até 3 turnos) |
| `RetrievedChunk` | Chunk retornado pelo Azure AI Search com score de similaridade e metadados completos |
| `PromptContext` | Conjunto de chunks selecionados que cabem no context budget (ADR-0002) |
| `AssistantResponse` | Resposta gerada com texto, citação de fonte e nível de confiança |
| `EscalationSignal` | Indicação de que o tema deve ser escalado, com canal específico e motivo |
| `SourceCitation` | Citação estruturada: nome do documento, versão, seção, data de atualização |
| `ConflictPresentation` | Apresentação estruturada de duas versões conflitantes com fonte para cada uma |

---

### Contexto 3 — Interface do Atendente

**Diretório:** `src/bot/` + `src/web/`  
**Specs:** `specs/teams-bot/` + `specs/painel-web/`  
**Arquivos-chave:** `bot.ts`, `response-card.ts`, `feedback-card.ts`, `App.tsx`

**Missão:** apresentar as respostas do assistente ao atendente no canal correto (Teams via Bot Framework ou painel web interno) e capturar a interação do atendente (leitura do trecho original, acionamento do feedback).

**Pertence a este contexto:**
- Renderização de Adaptive Cards no Teams com resposta + citação de fonte
- Formatação de respostas para o painel web interno
- Botão "Ver trecho original" e exibição do chunk bruto sem paráfrase
- Aviso visual para fontes informais (FAQ): `⚠️ Fonte não validada por Compliance`
- Card de escalada com canal específico e motivo
- Acionamento do fluxo de feedback (botão de reporte no card)
- Exibição automática do trecho original em respostas sobre temas de alto risco

**Não pertence a este contexto:**
- Lógica de busca ou geração de resposta (Contexto 2)
- Processamento do feedback após reporte (Contexto 4)

**Entidades centrais:**

| Entidade | Definição |
|---|---|
| `ResponseCard` | Adaptive Card com resposta formatada, SourceBadge e botão de feedback |
| `SourceBadge` | Componente visual com nome, versão, seção e data do documento fonte |
| `EscalationCard` | Card de escalada com canal (ramal, e-mail, área) e motivo da escalada |
| `InformalSourceWarning` | Aviso visual obrigatório quando a fonte é o FAQ informal |
| `FeedbackTrigger` | Ação do atendente que inicia o reporte de erro |

---

### Contexto 4 — Feedback e Auditoria

**Diretório:** `src/functions/feedback/`  
**Specs:** `specs/feedback-api/`  
**Arquivos-chave:** `handler.ts`, `validator.ts`

**Missão:** receber reportes de erro do atendente, persistir para investigação e fechar o loop conectando o reporte ao log da consulta original. Habilita o Gestor do Corpus a investigar e corrigir problemas na documentação.

**Pertence a este contexto:**
- Recebimento e validação do reporte de feedback
- Persistência do reporte com vínculo ao `QueryLog`
- Notificação ao Gestor do Corpus
- API de leitura de logs para investigação (painel web)

**Não pertence a este contexto:**
- A correção do documento em si (responsabilidade humana do Gestor do Corpus)
- A re-indexação após correção (Contexto 1)
- A geração da resposta que foi reportada como incorreta (Contexto 2)

**Entidades centrais:**

| Entidade | Definição |
|---|---|
| `FeedbackReport` | Reporte do atendente: descrição do problema, pergunta original, resposta recebida |
| `QueryLog` | Registro da consulta: timestamp, chunks recuperados com scores, resposta gerada |
| `CorpusAlert` | Notificação ao Gestor do Corpus com link ao FeedbackReport e QueryLog vinculados |

---

## Mapa de interação entre os contextos

```
┌─────────────────────────────────────────────────────────────┐
│  Contexto 1 — Ingestão                                      │
│  Document → Chunk → Index (Azure AI Search)                 │
│  + BlockedTopic list + ConflictRecord list                  │
└─────────────────────┬───────────────────────────────────────┘
                      │ chunks indexados + listas de controle
                      ▼
┌─────────────────────────────────────────────────────────────┐
│  Contexto 2 — Consulta e Resposta          (módulo principal)│
│  Query → RetrievedChunk → PromptContext                     │
│       → AssistantResponse | EscalationSignal                │
└────────────┬────────────────────────────────────┬───────────┘
             │ AssistantResponse / EscalationSignal│ QueryLog
             ▼                                     ▼
┌────────────────────────────┐     ┌───────────────────────────┐
│  Contexto 3 — Interface    │     │  Contexto 4 — Feedback    │
│  ResponseCard (Teams/web)  │────►│  FeedbackReport           │
│  EscalationCard            │     │  → CorpusAlert            │
└────────────────────────────┘     └───────────┬───────────────┘
                                               │ correção validada
                                               ▼
                                   Contexto 1 — re-indexação
```

---

## Fronteiras do assistente — o que faz e o que não faz

### O assistente FAZ

- Responder perguntas sobre SLAs, regras de frete e política de devolução **usando exclusivamente o corpus indexado**
- Citar a fonte de cada resposta (nome, versão, seção, data do documento)
- Detectar conflito entre chunks e apresentar **ambas as versões** sem escolher uma
- Sinalizar com aviso visual quando a fonte é o FAQ informal
- Orientar escalada com **canal específico** quando não tem resposta segura no corpus
- Disponibilizar o trecho original do documento ao atendente quando solicitado
- Solicitar esclarecimento do atendente em situações definidas (data do chamado para frete, tier do cliente para SLA, natureza da ocorrência para devolução vs sinistro)
- Registrar o reporte de feedback do atendente

### O assistente NÃO FAZ

- Acessar sistemas externos (portal do cliente, Azure DevOps, sistema de chamados)
- Abrir chamados, agendar coletas ou executar ações operacionais
- Negociar condições com o cliente ou propor exceções não documentadas
- Calcular valores usando conhecimento geral do LLM — apenas com dados presentes no corpus
- Responder sobre temas com cobertura bloqueada (PROC-043 em revisão, frete reverso em contratos pré-2023)
- Substituir a decisão humana em casos de ambiguidade documental não resolvida

### Sistemas externos — fronteira clara

| Sistema | Relação com o assistente |
|---|---|
| Azure AI Search | Fonte de chunks em runtime — o assistente busca, não grava |
| Azure OpenAI (GPT-4o) | Geração da resposta — o assistente orquestra a chamada |
| Microsoft Teams | Canal de entrega — interface do atendente (Contexto 3) |
| Portal do Cliente (portal.novatech.com.br) | O assistente orienta o atendente a direcionar o cliente para o portal, mas não integra |
| SharePoint / Confluence | Fonte de documentos para ingestão (Contexto 1), não para runtime |
| Azure DevOps (sistema de chamados) | Não integrado — o log de consultas é interno ao assistente |

---

*Leitura seguinte: `docs/onboarding.md` — seção 2 (Linguagem Ubíqua)*  
*Gerado na fase de estruturação — NovaTech × DB1*
