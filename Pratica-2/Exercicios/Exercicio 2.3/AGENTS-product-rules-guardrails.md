## Product Rules & Guardrails (Product Specialist)

> **Responsável:** Product Specialist  
> **Fonte primária:** `prompts/guardrails.md` — leia o documento completo antes de gerar código para o Contexto 2  
> **Esta seção** é um resumo operacional para consulta rápida durante o desenvolvimento.  
> Em caso de conflito entre esta seção e `prompts/guardrails.md`, o documento completo prevalece.

---

### Propósito do assistente

O assistente responde perguntas de atendentes da NovaTech sobre SLAs, regras de frete e política de devolução, usando exclusivamente o corpus indexado. Ele **não** executa ações, não acessa sistemas externos e não substitui decisão humana em casos de ambiguidade. Toda resposta é rastreável a um chunk de documento.

---

### Glossário de linguagem ubíqua

Agentes que geram código, testes, specs ou prompts para este projeto **devem usar os termos abaixo** — nunca sinônimos informais. Os termos do domínio são fonte de bugs silenciosos quando usados de forma inconsistente.

#### Entidades do domínio de negócio

| Termo canônico | Definição | Nunca usar |
|---|---|---|
| **chamado** | Registro formal de uma demanda do cliente no sistema de atendimento | "ticket", "solicitação", "ocorrência" |
| **tier** | Classificação do cliente: `Gold`, `Silver` ou `Standard`. Não existe `Platinum`. | "categoria", "nível", "tipo de cliente" |
| **cliente Gold** | Contrato anual acima de R$ 500.000 **ou** mais de 200 operações/mês. SLA de resposta: 2h úteis (chamado geral), 30 min (incidente crítico) | "cliente premium", "cliente VIP" |
| **cliente Silver** | Contrato entre R$ 100.000–500.000 **ou** 50–200 operações/mês. SLA de resposta: 4h úteis (geral), 1h (crítico) | "cliente médio" |
| **cliente Standard** | Todos os demais clientes. SLA de resposta: 8h úteis (geral), 2h (crítico) | "cliente comum", "cliente básico" |
| **SLA de resposta** | Tempo para o atendente dar o primeiro retorno ao cliente (mesmo que seja "estamos verificando") | "prazo de atendimento", "tempo de resposta" |
| **SLA de resolução** | Tempo para o problema ser efetivamente resolvido e o chamado encerrado | "prazo de fechamento", "tempo de resolução" |
| **incidente crítico** | Chamado que atende ≥1 critério: carga acima de R$ 100k com status desconhecido há +6h; carga perigosa com irregularidade; +5 chamados do mesmo cliente em 24h sobre o mesmo problema; risco à segurança de pessoas | "chamado urgente", "prioridade máxima" |
| **violação de SLA** | Situação em que o tempo de resposta ou resolução ultrapassou o limite contratual do tier | "SLA estourado", "prazo perdido" |
| **frete especial** | Modalidade de frete para cargas acima de 500 kg. Fórmula: Valor base × Multiplicador regional × Fator de peso | "frete pesado", "frete diferenciado" |
| **valor base** | Tarifa publicada mensalmente em `frete-base-AAAAMM.xlsx`. Atualizada pela Diretoria Comercial | "preço base", "tarifa" |
| **multiplicador regional** | Fator por região de destino aplicado sobre o valor base. Valores **divergem** entre PROC-042 v1 e v2 — sempre verificar `doc_status` antes de usar | "fator regional", "coeficiente" |
| **fator de peso** | Multiplicador por faixa de peso da carga. Valores **divergem** entre PROC-042 v1 (1.0/1.2/1.5) e v2 (1.0/1.15/1.4) | "coeficiente de peso" |
| **frete reverso** | Frete cobrado do cliente quando devolve por desistência (carga correta, sem defeito) | "frete de volta", "frete de devolução" |
| **desconto de volume** | Desconto automático: ≥8 fretes especiais/mês → 5%; ≥15 fretes/mês → 10% (PROC-042 v2). O FAQ informa limiar errado (10 fretes) — **ignorar** | "desconto de recorrência" |
| **carga especial** | Carga com peso acima de 500 kg, sujeita ao cálculo do PROC-042 | "frete especial" (para a carga) |
| **carga perigosa** | Carga classificada nas classes 1–6 da ANTT (Resolução ANTT nº 5.947/2021): explosivos (1), gases (2), líquidos inflamáveis (3), sólidos inflamáveis (4), oxidantes/peróxidos (5), substâncias tóxicas/infectantes (6). **Não elegível para devolução padrão.** | "produto perigoso", "carga ANTT" |
| **devolução** | Solicitação do cliente para devolver mercadoria **após entrega confirmada**. Regida pela POL-001 v3.1. Prazo: 7 dias úteis. Canal: Portal do Cliente | "retorno", "troca" |
| **sinistro** | Carga danificada **durante o transporte** (antes ou no ato da entrega). Prazo de registro: 48h. Canal: sinistros@novatech.com.br. Processo distinto da devolução — nunca confundir | "carga danificada" (como processo), "avaria" |
| **coleta reversa** | Operação de coleta da mercadoria no endereço do cliente para retorno ao CD. Agendada em até 2 dias úteis após aprovação | "busca", "retirada" |
| **CT-e** | Conhecimento de Transporte Eletrônico — documento obrigatório em todo chamado de devolução | "nota fiscal de frete" |
| **escalada** | Transferência da pergunta para o canal competente quando o assistente não tem resposta segura | "encaminhamento", "repasse" |
| **corpus** | Conjunto completo de chunks indexados no Azure AI Search que o assistente pode usar para responder | "base de dados", "base de conhecimento" |
| **chunk** | Unidade mínima de texto extraída de um documento e indexada com embedding | "trecho", "pedaço", "fragmento" |
| **documento normativo** | Documento com código (POL-XXX, PROC-XXX, SLA-XXXX) e responsável formal. Fonte de verdade. | "doc oficial", "arquivo" |
| **documento informal** | FAQ-Atendimento — sem validação por Compliance. Pode conter erros. Usar com restrições. | "FAQ", "anotações do time" |

#### Entidades do sistema (TypeScript — `src/shared/types.ts`)

| Tipo | Contexto | Definição |
|---|---|---|
| `Query` | Contexto 2 | Pergunta do atendente com histórico de até 3 turnos (6 mensagens) |
| `RetrievedChunk` | Contexto 2 | Chunk retornado pelo Azure AI Search com `confidenceScore` e metadados |
| `PromptContext` | Contexto 2 | Conjunto de chunks dentro do `contextBudget` (max 5 chunks) |
| `AssistantResponse` | Contexto 2 | Resposta gerada com `answer`, `sourceDocument`, `confidenceScore`, `escalationSignal`, `informalSourceWarning`, `conflictPresentation`, `queryId` |
| `SourceCitation` | Contexto 2 | `{ name, version, section, date }` — todos obrigatórios |
| `EscalationSignal` | Contexto 2 | `{ channel: EscalationChannel, reason: string }` |
| `ConflictPresentation` | Contexto 2 | `{ versionA: { value, source, informalSource }, versionB: { value, source, informalSource } }` |
| `DocumentStatus` | Contexto 1 | `"vigente" \| "histórico" \| "pendente_classificação" \| "vigente_transitório"` |
| `BlockedTopic` | Contexto 1→2 | Tema sem documento válido no corpus; handler escala sem chamar o LLM |
| `QueryLog` | Contexto 2→4 | Registro auditável da consulta: chunks recuperados, scores, attempt count, resposta |
| `confidenceScore` | `number` 0–1 | Score do chunk mais relevante. Threshold: `CONFIDENCE_THRESHOLD` (default `0.75`) |

---

### Regras de comportamento do assistente

As regras abaixo governam o comportamento em runtime. Cada uma tem **enforcement** — via código (determinístico, 100%) ou via prompt (probabilístico, verificado por VC de regressão). Agentes que geram código para o Contexto 2 devem garantir que o enforcement via código esteja implementado antes de submeter.

#### DEVE — comportamentos obrigatórios

**G-D01 · Citar fonte completa em toda resposta não-escalada** `enforcement: código`  
`sourceDocument` deve ter `name`, `version`, `section` e `date` — todos preenchidos. O `response-validator.ts` rejeita com HTTP 422 se qualquer campo estiver ausente. Nunca citar apenas o nome do documento sem a versão. *(Previne INC-02)*

**G-D02 · Filtrar chunks por `doc_status` antes do prompt** `enforcement: código`  
No `handler.ts`, após o retorno do Azure AI Search, descartar qualquer chunk com `doc_status` diferente de `"vigente"` ou `"vigente_transitório"`. Chunks descartados vão para o `QueryLog` com `discarded_reason`. O `prompt-builder.ts` nunca recebe chunk histórico. *(Previne INC-02)*

**G-D03 · Responder em português formal** `enforcement: prompt`  
Sem gírias, abreviações ou mistura de idiomas. Tom de especialista interno. VC de regressão: pergunta em inglês → resposta em português com orientação de idioma.

**G-D04 · Emitir `EscalationSignal` com canal específico** `enforcement: código`  
Quando sem resposta segura, o campo `channel` deve ser um dos quatro valores do enum `EscalationChannel`. O schema Zod rejeita qualquer string fora desse conjunto. *(Previne INC-03)*

```typescript
// src/shared/types.ts
export const EscalationChannel = z.enum([
  "ramal 4500 — Gestão de Riscos",   // carga perigosa, PROC-043 bloqueado
  "Comercial",                        // conflito sem resolução, frete reverso pré-2023
  "sinistros@novatech.com.br",        // sinistro — carga danificada em trânsito
  "supervisor de atendimento",        // tema não coberto, fora do escopo
]);
```

**G-D05 · Verificar `BlockedTopic` como primeira operação do handler** `enforcement: código`  
Antes de qualquer chamada ao Azure AI Search ou ao LLM. Se tema bloqueado detectado: emitir `EscalationSignal` imediatamente. Tópicos atualmente bloqueados:

| Tema | Canal |
|---|---|
| Frete de carga perigosa acima de 500 kg (PROC-043 em revisão) | `"ramal 4500 — Gestão de Riscos"` |
| Custo de frete reverso em contratos anteriores a 01/12/2023 | `"Comercial"` |
| Interceptação de carga em trânsito (PROC-088 ausente) | `"supervisor de atendimento"` |

#### NÃO DEVE — comportamentos proibidos

**G-N01 · Nunca aplicar regra geral sem verificar exceções do mesmo documento** `enforcement: prompt + VC`  
Se um chunk contém uma regra geral (ex: "prazo de 7 dias"), o assistente deve verificar os chunks de exceção do mesmo documento antes de responder. Se os chunks de exceção não foram recuperados mas a pergunta envolve um caso possivelmente especial, declarar limitação e escalar. VC obrigatório: pergunta sobre devolução de carga perigosa → deve citar seção 3.2, nunca o prazo de 7 dias. *(Causa-raiz do INC-01)*

**G-N02 · Nunca usar chunk com `doc_status = "histórico"` como fonte** `enforcement: código`  
Idêntico ao G-D02 pelo ângulo proibitivo. Chunks históricos podem ter score alto — o filtro determinístico no `handler.ts` é a única garantia. *(Causa-raiz do INC-02)*

**G-N03 · Nunca inventar valor, prazo ou regra** `enforcement: prompt + VC`  
Sem inferência, extrapolação ou uso de conhecimento geral do LLM. Se a informação não está explicitamente no chunk, declarar ausência e escalar. VC obrigatório: pergunta sobre tema ausente do corpus → resposta deve ser escalada, não estimativa. *(Previne INC-01, INC-03)*

**G-N04 · Nunca escolher entre versões conflitantes** `enforcement: código`  
Quando o handler detecta conflito (via lista `ConflictRecord` ou heurística de `doc_codigo`), o `response-validator.ts` rejeita resposta com valor único onde deveria haver `ConflictPresentation`. *(Previne INC-02)*

**G-N05 · Nunca responder sobre carga perigosa apenas com FAQ** `enforcement: prompt + código`  
Se todos os chunks recuperados para tema de carga perigosa têm `documentClassification = "informal"`, o handler força escalada para `"ramal 4500 — Gestão de Riscos"` sem chamar o LLM. *(Previne INC-01)*

**G-N06 · Nunca declarar ausência sem dois attempts de busca** `enforcement: código`  
Se `maxScore(chunks) < CONFIDENCE_THRESHOLD` no primeiro attempt, o handler reformula a query e tenta novamente. Só após o segundo attempt com score ainda baixo emite escalada. `QueryLog` registra `searchAttempts: 1 | 2`. *(Causa-raiz do INC-03)*

**G-N07 · Nunca tratar devolução e sinistro como o mesmo processo** `enforcement: prompt + VC`  
Gatilhos de esclarecimento: "carga danificada", "avaria", "produto com defeito", "embalagem violada", "lacre rompido", "carga molhada", "durante o transporte", "chegou quebrada". Antes de orientar, perguntar: "a carga foi danificada em trânsito ou o cliente quer devolver após receber?"

#### QUANDO EM DÚVIDA — comportamentos de fallback

**G-F01 · Versão de documento ambígua → pedir data do chamado** `enforcement: prompt`  
Para frete especial com PROC-042 v1 e v2 disponíveis: solicitar data do chamado. v1 = chamados anteriores a 01/12/2023; v2 = chamados a partir de 01/12/2023. Se atendente não responder no turno seguinte: usar v2 com aviso explícito.

**G-F02 · Dúvida sobre classificação de carga perigosa → escalar** `enforcement: código`  
Se o assistente não consegue determinar com certeza se uma carga é perigosa (ANTT classes 1–6), assumir que é perigosa e encaminhar ao ramal 4500. Conservadorismo deliberado — custo de escalada desnecessária é baixo; custo de orientação incorreta é regulatório.

**G-F03 · Conflito sem escopo temporal → apresentar ambas as versões** `enforcement: código`  
Se dois chunks conflitantes não têm `doc_escopo_temporal` definido: `ConflictPresentation` com ambas as versões + escalada ao `"Comercial"`. O `response-validator.ts` garante isso.

**G-F04 · Sem resposta após dois attempts → declarar ausência específica** `enforcement: prompt`  
Não usar mensagem genérica. Formato obrigatório: *"Não encontrei informação sobre [tema específico] no corpus atual. Encaminhe ao [canal]."* O tema específico ajuda o Gestor do Corpus a identificar gaps de cobertura.

---

### Restrições que impactam geração de código

Agentes que geram código para `src/functions/query/` ou `src/services/` devem respeitar as seguintes restrições. Violações são detectadas no code review.

#### Schema obrigatório da resposta

Todo código que produz `AssistantResponse` deve satisfazer:

```typescript
// Invariantes verificadas pelo response-validator.ts — nunca contornar
//
// 1. sourceDocument obrigatório quando escalationSignal = null
if (!response.escalationSignal && !response.sourceDocument) {
  throw new ValidationError("G-D01: sourceDocument ausente em resposta não-escalada");
}

// 2. sourceDocument deve ter todos os quatro campos
if (response.sourceDocument) {
  const { name, version, section, date } = response.sourceDocument;
  if (!name || !version || !section || !date) {
    throw new ValidationError("G-D01: SourceCitation incompleta");
  }
}

// 3. escalationSignal = null e sourceDocument = null simultaneamente é inválido
if (!response.escalationSignal && !response.sourceDocument) {
  throw new ValidationError("resposta sem fonte e sem escalada — estado inválido");
}

// 4. conflictPresentation implica sourceDocument = null
if (response.conflictPresentation && response.sourceDocument) {
  throw new ValidationError("G-N04: conflito detectado mas sourceDocument preenchido");
}

// 5. informalSourceWarning = true quando qualquer chunk fonte é informal
// (verificado no handler antes do prompt-builder)
```

#### Ordem de operações no `handler.ts`

O handler deve seguir esta sequência — qualquer reordenação viola guardrails:

```
1. Validar input (Zod) → HTTP 400 se inválido
2. Verificar BlockedTopic (G-D05) → EscalationSignal imediato se bloqueado
3. Buscar chunks (attempt 1)
4. Filtrar por doc_status (G-D02) — descartar "histórico" e "pendente_classificação"
5. Se maxScore < threshold: reformular e buscar (attempt 2) (G-N06)
6. Se ainda < threshold: EscalationSignal (G-D04)
7. Detectar conflito entre chunks (G-N04)
8. Verificar documentClassification de todos os chunks (G-N05)
9. Construir PromptContext (respeitar contextBudget — ADR-0002)
10. Chamar LLM (completion.ts)
11. Validar resposta (response-validator.ts)
12. Emitir QueryLog (fire-and-forget)
13. Retornar AssistantResponse
```

#### Proibições de implementação

- `console.log`, `console.error`, `console.warn` em qualquer arquivo de produção → usar `pino` (`src/shared/logger.ts`)
- Importações cruzadas entre contextos: `src/functions/query/` não importa de `src/pipeline/`, `src/bot/` ou `src/functions/feedback/`
- `any` implícito em TypeScript — `strict: true` no `tsconfig.json`
- Validação manual de tipos via `typeof` ou `instanceof` nas bordas — usar Zod
- Hardcode de valores de configuração (`CONFIDENCE_THRESHOLD`, `MAX_CHUNKS`, `CONTEXT_BUDGET_*`) — usar variáveis de ambiente via `src/shared/config.ts`
- Chunks com `doc_status = "histórico"` no `PromptContext` — nunca, sob nenhuma circunstância

---

### Referências a documentos de spec no repositório

Antes de implementar qualquer funcionalidade do Contexto 2 (query endpoint), leia nesta ordem:

| Documento | Localização | O que contém |
|---|---|---|
| Bounded Contexts | `docs/onboarding.md` seção 1 | Fronteiras entre os 4 contextos; o que pertence e não pertence ao Contexto 2 |
| Linguagem Ubíqua | `docs/onboarding.md` seção 2 | Termos canônicos do domínio (fonte desta seção do AGENTS.md) |
| Guardrails completos | `prompts/guardrails.md` | Documento de referência desta seção; inclui exemplos ❌/✅, trechos de prompt e VCs de regressão |
| Requirements do query endpoint | `specs/query-endpoint/requirements.md` | Outcomes, scope boundaries, prior decisions (ADRs), constraints e 14 VCs |
| System prompt | `prompts/system-prompt.md` | System prompt atual — contém os trechos de prompt dos guardrails G-D03, G-N01, G-N03, G-N05, G-N07, G-F04 |
| Tipos compartilhados | `src/shared/types.ts` | Fonte de verdade dos tipos TypeScript do Contexto 2 |
| ADR-0001 | `docs/adr/0001-modelo-gpt4o.md` | Decisão de modelo: Azure OpenAI GPT-4o |
| ADR-0002 | `docs/adr/0002-context-budget.md` | Context budget: ~4K system prompt + ~8K chunks + pergunta + 3 turnos |
| ADR-0003 | `docs/adr/0003-documentos-contraditorios.md` | Tratamento de conflito: metadado + instrução no prompt |
| ADR-0004 | `docs/adr/0004-chunking-tabelas.md` | Chunking de tabelas como unidade atômica |
| Corpus de referência | `data/retrieval-corpus/chunks-novatech.md` | Chunks simulados para testes locais (substitui Azure AI Search) |
| Golden queries | `prompts/eval/golden-queries.json` | Perguntas de referência para VCs de regressão dos guardrails via prompt |

> ⚠️ Os arquivos `docs/adr/000N-*.md` ainda não foram criados no repositório. O conteúdo de cada ADR está inline na seção "Prior decisions" de `specs/query-endpoint/requirements.md`. Criar os ADRs é responsabilidade do Tech Lead e deve ocorrer antes do início da implementação.
