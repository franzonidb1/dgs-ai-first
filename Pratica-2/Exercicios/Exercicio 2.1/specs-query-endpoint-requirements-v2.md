# requirements.md — Query Endpoint

> **Localização canônica no repo:** `specs/query-endpoint/requirements.md`
> **Versão:** 2.0 — resolve ambiguidades identificadas no Tech Lead Review
> **Contexto:** Contexto 2 — Consulta e Resposta (`docs/onboarding.md`, seção 1)
> **Formato:** Spec Driven Development (SDD)
> **Responsável pela spec:** Product Specialist
> **Implementação:** `src/functions/query/` + `src/services/`
> **Leitura obrigatória antes desta spec:** `docs/onboarding.md` (Bounded Contexts + Linguagem Ubíqua)

### Changelog v1 → v2

| ID | Ambiguidade resolvida | Seção alterada |
|---|---|---|
| B-01 | Definição formal de "turno"; turnos de esclarecimento não consomem histórico | ADR-0002 |
| B-02 | Algoritmo de detecção de conflito: dois mecanismos em ordem de precedência | ADR-0003 |
| B-03 | `BlockedTopic` via lookup no handler, fora do system prompt | ADR-0002 + ADR-0003 |
| B-04 | Threshold aplicado por chunk; `confidenceScore` = mínimo dos chunks incluídos | C-P02 |
| B-05 | `QueryLog` criado pelo Contexto 2, emitido via fire-and-forget assíncrono | Scope boundaries |
| C-01 | Lista explícita de gatilhos de esclarecimento devolução/sinistro | C-P06 |
| C-02 | Política de descarte de esclarecimento pendente; novo VC-13 | VC-13 (novo) |
| C-03 | `ConflictPresentation` com campo `informalSource` por versão; novo VC-14 | VC-10 schema + VC-14 (novo) |
| C-04 | Completude de tabela via metadado `chunk_is_complete_table`, não heurística | ADR-0004 |
| C-05 | Threshold configurável via `CONFIDENCE_THRESHOLD`; processo de calibração | C-P02 + Outcomes |
| C-06 | Métrica de alucinação movida para eval manual; métricas automáticas corrigidas | Seção 1 — Outcomes |
| E-01 | Schema Zod do request (`QuerySchema`) adicionado ao VC-10 | VC-10 |
| E-02 | Aviso de fonte informal removido do `answer`; responsabilidade do Contexto 3 | VC-07 |
| E-03 | Aviso sobre ADRs inexistentes adicionado à seção 7 | Seção 7 |

---

---

## 1. Outcomes

O query endpoint existe para que um atendente da NovaTech consiga obter, em menos de 30 segundos, uma resposta fundamentada na documentação oficial — sem abrir SharePoint, Confluence ou pasta de rede. É o único componente do sistema que entrega valor diretamente ao atendente em tempo real.

### O que o sucesso parece

**Para o atendente:**
- Digita uma pergunta em linguagem natural e recebe uma resposta com a fonte identificada (nome do documento, versão, seção, data) — suficiente para verificar sem esforço adicional.
- Quando há conflito documental (ex: PROC-042 v1 e v2 com multiplicadores diferentes), vê as duas versões com suas fontes — nunca uma escolha arbitrária do modelo.
- Quando o assistente não tem resposta segura, recebe um canal de escalada específico — nunca fica sem direção.

**Para o time:**
- Toda resposta gerada é rastreável a um chunk indexado. Nenhuma informação vem do conhecimento geral do LLM.
- Qualquer resposta incorreta pode ser investigada via `QueryLog` — os chunks recuperados e seus scores ficam registrados.

### Métricas de sucesso — verificáveis no go-live

| Métrica | Alvo | Limite aceitável | Verificação |
|---|---|---|---|
| Latência P95 (pergunta → resposta completa) | ≤ 15s | ≤ 30s | Automática — APM |
| Latência P99 | ≤ 30s | ≤ 60s | Automática — APM |
| Respostas não-escaladas com `sourceDocument` preenchido | 100% | — | Automática — schema Zod |
| Respostas com `escalationSignal = null` e `sourceDocument = null` simultaneamente | 0% | — | Automática — schema Zod |
| Escaladas com `escalationSignal.channel` específico | 100% | — | Automática — schema Zod |
| Respostas com `informalSourceWarning` ausente quando fonte é FAQ | 0% | — | Automática — testes de contrato |
| Respostas com conteúdo não rastreável ao corpus (alucinação) | 0% | — | Manual — eval semanal com `prompts/eval/golden-queries.json` nos primeiros 30 dias; mensal após estabilização |

---

## 2. Scope boundaries

Os scope boundaries derivam diretamente do Contexto 2 definido em `docs/onboarding.md` (seção 1 — Bounded Contexts). O Contexto 2 tem missão única: receber `Query`, produzir `AssistantResponse` ou `EscalationSignal`. Tudo fora dessa transformação pertence a outro contexto.

### Dentro do escopo desta spec

O que pertence ao Contexto 2, conforme `docs/onboarding.md`:

- Validação e normalização da `Query` recebida (Zod)
- Busca de `RetrievedChunk` no Azure AI Search com `confidenceScore`
- Verificação da lista de `BlockedTopic` (produzida pelo Contexto 1) antes de usar chunks
- Detecção de `ConflictPresentation` nos chunks recuperados
- Construção do `PromptContext` dentro do `contextBudget` (ADR-0002)
- Chamada ao Azure OpenAI GPT-4o (ADR-0001) via `completion.ts`
- Validação da `AssistantResponse` gerada contra os guardrails de produto
- Construção da `SourceCitation` obrigatória em toda resposta não-escalada
- Determinação do `EscalationSignal` com canal específico quando sem resposta segura
- Emissão do `QueryLog` para consumo pelo Contexto 4: o `handler.ts` cria o objeto `QueryLog` imediatamente antes de retornar a resposta ao Contexto 3, e o emite de forma assíncrona (fire-and-forget via Azure Service Bus ou equivalente configurado em `plan.md`). A indisponibilidade do Contexto 4 não bloqueia a resposta ao atendente — o `handler.ts` loga o erro de emissão via `pino` com nível `warn` e continua. O `QueryLog` inclui chunks descartados (por `BlockedTopic` ou por threshold) com flag `included_in_prompt: boolean` para cada chunk recuperado.

### Fora do escopo desta spec

O que pertence a outros contextos e não deve aparecer em código ou testes do Contexto 2:

| O que | Pertence a | Spec de referência |
|---|---|---|
| Indexação de documentos no Azure AI Search | Contexto 1 — Ingestão | `specs/pipeline-ingestao/requirements.md` |
| Manutenção da lista de `BlockedTopic` | Contexto 1 — Ingestão | `specs/pipeline-ingestao/requirements.md` |
| Renderização de `ResponseCard` no Teams | Contexto 3 — Interface | `specs/teams-bot/requirements.md` |
| Exibição do `InformalSourceWarning` visual | Contexto 3 — Interface | `specs/teams-bot/requirements.md` |
| Persistência do `FeedbackReport` | Contexto 4 — Feedback | `specs/feedback-api/requirements.md` |
| Autenticação do atendente | Infraestrutura | `specs/query-endpoint/plan.md` |
| Ciclo de vida dos documentos no índice | Contexto 1 — Ingestão | `specs/pipeline-ingestao/requirements.md` |

**Regra de violação de boundary:** se um arquivo em `src/functions/query/` ou `src/services/` importar qualquer módulo de `src/pipeline/`, `src/bot/` ou `src/functions/feedback/`, é uma violação de boundary. Os contextos se comunicam por contratos de interface (tipos TypeScript em `src/shared/types.ts`), nunca por importação direta.

---

## 3. Prior decisions

As decisões abaixo foram tomadas na fase anterior (Cenário 1) e estão fechadas. Esta spec não as reabre. Se surgir razão técnica para questionar qualquer uma delas durante a implementação, o processo correto é criar um novo ADR em `docs/adr/` e submeter para revisão do Tech Lead — nunca alterar o comportamento silenciosamente.

### ADR-0001 — Modelo: Azure OpenAI GPT-4o

**Decisão:** o modelo de linguagem do assistente é o GPT-4o via Azure OpenAI Service.

**O que está fechado para esta spec:**
- Nenhum outro modelo pode ser usado em produção (nem mesmo outras versões do GPT sem revisão do ADR)
- A integração usa o SDK oficial do Azure, não chamadas HTTP diretas
- A janela de 128K tokens é o limite absoluto; o `contextBudget` deve sempre ficar abaixo desse limite com margem de segurança

**Implicação para implementação:** `completion.ts` usa `@azure/openai` ou `openai` com configuração de endpoint Azure. A escolha do SDK exato fica em `specs/query-endpoint/plan.md`.

### ADR-0002 — Estratégia de contexto: context budget fixo

**Decisão:** o `contextBudget` é distribuído da seguinte forma:

| Slot | Tamanho aproximado | Conteúdo |
|---|---|---|
| System prompt | ~4.000 tokens | Instruções fixas do assistente e guardrails por versão |
| Chunks | ~8.000 tokens | Máximo 5 `RetrievedChunk` × ~1.600 tokens cada |
| Pergunta do atendente | ~200 tokens | A `Query` atual |
| Histórico de sessão | ~2.000 tokens | Últimos 3 turnos da conversa |
| **Total** | **~14.200 tokens** | Margem ampla abaixo do limite de 128K |

**Definição formal de turno:** um turno é um par composto por uma mensagem do atendente e a resposta do assistente. O limite de 3 turnos equivale a 6 mensagens no `conversationHistory`. Turnos de esclarecimento (pedido de dado + resposta do atendente) contam como um turno lógico único — eles compõem internamente a resolução de uma única `Query` e não consomem o histórico de sessão.

**O que está fechado para esta spec:**
- O `prompt-builder.ts` deve garantir que o prompt nunca exceda o budget por slot, não apenas o total
- Máximo de 5 chunks por consulta — mesmo que a busca retorne mais resultados com score alto
- Histórico limitado a 3 turnos (6 mensagens) — turnos mais antigos são descartados, não resumidos
- O system prompt é fixo por versão de deploy — não é gerado dinamicamente em runtime
- A lista de `BlockedTopic` é verificada via lookup antes da busca de chunks, fora do system prompt — ver B-03 abaixo

**Implicação para implementação:** `search.ts` retorna os 5 chunks com maior `confidenceScore`. `prompt-builder.ts` monta o prompt na ordem: system prompt → chunks (do mais ao menos relevante) → histórico → pergunta. O lookup de `BlockedTopic` ocorre no `handler.ts` antes de qualquer chamada ao `search.ts`.

### ADR-0003 — Documentos contraditórios: metadado + instrução no prompt

**Decisão:** o tratamento de documentos contraditórios usa dois mecanismos combinados:

1. **No pipeline (Contexto 1):** cada `Chunk` carrega o metadado `doc_status` do seu documento de origem. O par conflitante PROC-042 v1 / v2 é registrado como `ConflictRecord` com escopo temporal (v1 para chamados anteriores a 01/12/2023, v2 para chamados a partir dessa data).

2. **No prompt (Contexto 2):** o system prompt instrui explicitamente o modelo a nunca escolher entre versões conflitantes — deve apresentar ambas com suas fontes e orientar o atendente a confirmar com base na data do chamado.

**Algoritmo de detecção de conflito — dois mecanismos em ordem de precedência:**

**Mecanismo 1 (precedência maior) — lista estática de `ConflictRecord`:** o Contexto 1 produz e mantém uma lista de pares de documentos conhecidos como conflitantes, com o tema de conflito. O `handler.ts` carrega essa lista no início de cada request. Quando dois chunks cujos `doc_codigo` constam em um `ConflictRecord` são recuperados para a mesma pergunta, o sistema declara conflito — sem análise semântica.

**Mecanismo 2 (fallback) — heurística por metadado:** se dois chunks têm o mesmo `doc_codigo` e versões diferentes, e ambos têm `doc_status = vigente` ou `vigente_transitório`, o sistema declara conflito automaticamente.

O Mecanismo 1 é aplicado primeiro. Se não houver `ConflictRecord` correspondente, o Mecanismo 2 é aplicado.

**Mecanismo de `BlockedTopic` em runtime:** a lista de `BlockedTopic` é carregada pelo `handler.ts` a partir de um lookup externo (variável de ambiente ou configuração) antes da busca de chunks — não está embutida no system prompt. O system prompt é fixo por versão de deploy. Quando um novo documento resolve um tema bloqueado (ex: PROC-043 é publicado), a lista de `BlockedTopic` é atualizada na configuração sem necessidade de redeploy do system prompt.

**O que está fechado para esta spec:**
- O Contexto 2 não resolve conflitos — apresenta-os via `ConflictPresentation`
- A escolha da versão correta é do atendente (com apoio da data do chamado como critério objetivo)
- Documentos com `doc_status = histórico` nunca devem aparecer em `RetrievedChunk` para o Contexto 2 — essa filtragem é responsabilidade do índice (Contexto 1)
- A verificação de `BlockedTopic` ocorre no `handler.ts` antes do `search.ts` — não no `response-validator.ts`

**Implicação para implementação:** `response-validator.ts` deve rejeitar respostas que contenham apenas uma versão quando o `handler.ts` sinalizou conflito detectado pelos dois mecanismos acima. O sinal de conflito detectado é passado como parâmetro interno entre o handler e o validator — não re-detectado pelo validator.

### ADR-0004 — Chunking especial para tabelas

**Decisão:** tabelas nos documentos (como a tabela de multiplicadores regionais do PROC-042 e a tabela de SLAs) devem ser tratadas como unidades atômicas no chunking — nunca quebradas entre dois chunks. O protótipo open-source (ChromaDB + sentence-transformers) quebrou tabelas e gerou respostas com dados incompletos.

**O que está fechado para esta spec:**
- O `search.ts` assume que chunks de tabelas chegam completos do índice
- A completude de um chunk de tabela é indicada pelo metadado `chunk_is_complete_table: boolean`, produzido pelo Contexto 1 durante o chunking. O `response-validator.ts` lê esse metadado — não aplica heurística própria de detecção
- Se `chunk_is_complete_table = false`, o chunk é descartado do `PromptContext` e o fato é registrado no `QueryLog` como alerta de falha de ingestão
- A solução de chunking de tabelas fica em `specs/pipeline-ingestao/` (Contexto 1)

**Implicação para implementação:** `response-validator.ts` verifica o campo `chunk_is_complete_table` nos metadados de cada `RetrievedChunk` antes de incluí-lo no `PromptContext`. Chunks com `chunk_is_complete_table = false` são logados via `pino` com nível `warn` e descartados silenciosamente — nunca usados como base de `SourceCitation`.

---

## 4. Constraints

### Constraints de produto — derivadas da spec de RAG do Cenário 1

Estas constraints governam o comportamento do assistente e não podem ser relaxadas sem aprovação do Product Specialist:

**C-P01 — Rastreabilidade obrigatória**
Toda `AssistantResponse` não-escalada deve ter `sourceDocument` preenchido com `SourceCitation` completa (nome, versão, seção, data). A ausência de qualquer um desses campos é uma violação — não um campo opcional.

**C-P02 — Proibição de conhecimento geral**
O assistente nunca usa o conhecimento geral do GPT-4o como substituto de informação ausente no corpus. O threshold de confiança é `0.75` (configurável via variável de ambiente `CONFIDENCE_THRESHOLD`, default `0.75` — deve ser revisado após 30 dias com base nos dados do `QueryLog`).

**Regras de aplicação do threshold:**
- Para respostas de fonte única: se o chunk top-1 tem `confidenceScore < 0.75`, a resposta é uma escalada orientada.
- Para respostas multi-chunk (perguntas que cruzam categorias): o threshold é aplicado **por chunk individualmente** — cada chunk incluído no `PromptContext` deve ter `confidenceScore ≥ 0.75`. Chunks abaixo do threshold são descartados antes da construção do prompt.
- Se após o descarte de chunks abaixo do threshold restar pelo menos um chunk válido para cada sub-tema da pergunta: responder com os chunks disponíveis e citar as fontes correspondentes.
- Se após o descarte restar zero chunks válidos para algum sub-tema: o `confidenceScore` reportado na resposta é o do chunk mais relevante descartado, e `escalationSignal` é emitido para esse sub-tema.
- O `confidenceScore` reportado na `AssistantResponse` é o **mínimo** entre os scores dos chunks efetivamente incluídos no `PromptContext` — representa o "elo mais fraco" da resposta.

**C-P03 — Verificação de `BlockedTopic` tem precedência**
A verificação da lista de `BlockedTopic` ocorre antes de qualquer uso dos chunks recuperados — não depois. Mesmo que chunks relacionados ao tema bloqueado tenham sido recuperados com score alto, a resposta deve ser uma escalada com `escalationSignal.channel` específico.

**C-P04 — Conflito não é resolvido, é apresentado**
Quando dois chunks com `doc_status = vigente_transitório` cobrem o mesmo tema com valores divergentes, a resposta deve ser uma `ConflictPresentation` — nunca uma escolha. O `response-validator.ts` deve rejeitar respostas que resolvam o conflito silenciosamente.

**C-P05 — Fonte informal gera aviso, nunca silêncio**
Chunks com `documentClassification = informal` (FAQ-Atendimento) podem compor respostas, mas a `AssistantResponse` deve ter `informalSourceWarning = true`. O Contexto 3 usa esse campo para exibir o aviso visual — mas a responsabilidade de emitir o campo é do Contexto 2.

**C-P06 — Distinção devolução/sinistro é obrigatória antes de responder**
Quando a pergunta contém qualquer uma das expressões ou intenções listadas abaixo, o endpoint deve solicitar esclarecimento sobre a natureza antes de responder — devolução (POL-001, 7 dias, Portal) ou sinistro (48h, sinistros@novatech.com.br):

**Gatilhos de esclarecimento (detecção semântica pelo LLM com as seguintes referências):**
- Expressões diretas: "carga danificada", "chegou danificada", "avaria", "produto com defeito", "embalagem violada", "lacre rompido", "carga molhada", "carga quebrada", "mercadoria avariada"
- Expressões de estado pós-entrega: "cliente recebeu e", "após a entrega", "depois que chegou"
- Expressões de ocorrência em trânsito: "durante o transporte", "antes de chegar", "no caminho", "na entrega"

O LLM classifica a intenção da pergunta contra essa lista antes de buscar chunks. Se a intenção for ambígua entre devolução e sinistro, solicita esclarecimento — uma única pergunta, nunca duas em sequência. Responder com um único fluxo sem essa distinção é uma violação da C-P06.

**C-P07 — Escalada tem canal específico, nunca genérico**
Todo `EscalationSignal` deve ter `channel` preenchido com um dos valores definidos na tabela de canais de escalada (seção 5 deste documento). A string "falar com o supervisor" sem identificação do canal específico é inaceitável.

### Constraints de stack — derivadas do Anexo D (starter repo)

**C-S01** — Linguagem: TypeScript ESM, `strict: true`, target ES2022. Nenhum `any` implícito.

**C-S02** — Validação de input e output: Zod em todas as bordas do módulo. Sem validação manual de tipos via `typeof` ou `instanceof`.

**C-S03** — Logging: `pino` exclusivamente. Nenhum `console.log`, `console.error` ou `console.warn` em código de produção.

**C-S04** — Padrão de função: Azure Functions v4. O handler é exportado como função nomeada — não como default export.

**C-S05** — Cobertura de testes: ≥ 80% de linhas, conforme `vitest.config.ts`. A cobertura é medida por módulo (`search.ts`, `prompt-builder.ts`, `completion.ts`, `response-validator.ts`, `handler.ts`) — não apenas pelo total agregado.

**C-S06** — Sem dependências circulares entre serviços. `handler.ts` orquestra; os serviços individuais não se importam entre si.

---

## 5. Verification criteria

Cada critério é um caso de teste que o QA executa sem pedir esclarecimentos. O formato é: **Dado [estado do sistema] / Quando [ação] / Então [resultado verificável]**.

Os critérios cobrem os quatro cenários de resposta possíveis: (A) resposta com fonte, (B) conflito documental, (C) escalada por tópico bloqueado, (D) escalada por ausência de resposta.

---

### VC-01 — Resposta com fonte para pergunta coberta pelo corpus

**Cenário:** pergunta simples, fonte única, sem conflito.

**Dado** que o índice contém o chunk `SLA-2024-C` com `documentStatus = vigente`, `documentClassification = normativo`, e o texto inclui os SLAs para incidentes críticos por tier  
**Quando** o atendente envia: `"qual o prazo de resposta para cliente Gold em incidente crítico?"`  
**Então:**
- HTTP 200
- `answer` contém "30 minutos"
- `sourceDocument.name = "SLA-2024"`
- `sourceDocument.version = "2024.1"`
- `sourceDocument.section = "seção 2"`
- `sourceDocument.date = "02/01/2024"`
- `confidenceScore ≥ 0.75`
- `escalationSignal = null`
- `informalSourceWarning = false`
- `conflictPresentation = null`
- `queryId` é UUID v4 válido
- Latência total ≤ 15 segundos

---

### VC-02 — Esclarecimento solicitado antes de responder sobre frete com versão ambígua

**Cenário:** pergunta de frete especial sem data do chamado — campo obrigatório para determinar a versão do PROC-042 aplicável.

**Dado** que o índice contém `PROC-042-A` (v1, fator 1.2) e `PROC-042v2-A` (v2, fator 1.15), ambos com `documentStatus = vigente_transitório`  
**Quando** o atendente envia: `"qual o fator de peso para uma carga de 2.000 kg?"` sem informar data do chamado  
**Então:**
- HTTP 200
- `answer` contém uma pergunta de esclarecimento sobre a data do chamado
- `answer` **não** contém nenhum valor numérico de fator de peso
- `escalationSignal = null` (não é escalada — é pedido de esclarecimento)
- `conflictPresentation = null` (conflito ainda não apresentado — aguardando dado)
- `sourceDocument = null`

**E quando** o atendente responde com data `"15/01/2024"` (posterior a 01/12/2023)  
**Então:**
- `answer` contém o fator 1.15 (v2)
- `sourceDocument.name = "PROC-042-v2"`, `sourceDocument.version = "2.0"`
- `conflictPresentation = null` (sem conflito após data informada)

**E quando** o atendente responde com data `"01/10/2023"` (anterior a 01/12/2023)  
**Então:**
- `answer` contém o fator 1.2 (v1)
- `sourceDocument.name = "PROC-042"`, `sourceDocument.version = "1.0"`

---

### VC-03 — Conflito apresentado quando data não resolve a ambiguidade

**Cenário:** pergunta sobre aspecto do PROC-042 que não tem escopo temporal claro nos metadados — o sistema apresenta ambas as versões.

**Dado** que o índice contém `PROC-042-C` (v1, prazo +2 dias) e `PROC-042v2-C` (v2, prazo +3 dias), sem `doc_escopo_temporal` definido para esse campo específico  
**Quando** o atendente envia: `"qual o prazo adicional para frete especial?"`  
**Então:**
- HTTP 200
- `conflictPresentation` não é null
- `conflictPresentation.versionA.value = "prazo padrão + 2 dias úteis"`
- `conflictPresentation.versionA.source.name = "PROC-042"`, `version = "1.0"`
- `conflictPresentation.versionB.value = "prazo padrão + 3 dias úteis"`
- `conflictPresentation.versionB.source.name = "PROC-042-v2"`, `version = "2.0"`
- `answer` instrui o atendente a confirmar com o Comercial qual versão aplica ao chamado
- `escalationSignal.channel = "Comercial"`
- `sourceDocument = null` (não há fonte única — há duas)

---

### VC-04 — Escalada obrigatória para tópico bloqueado, ignorando chunks recuperados

**Cenário:** tópico bloqueado — PROC-043 em revisão. Mesmo com chunks relacionados no índice, o sistema não deve tentar responder.

**Dado** que "frete de carga perigosa acima de 500 kg" está na lista de `BlockedTopic`  
**Dado** que o índice contém `PROC-042v2-A` com menção a "cargas perigosas seguem PROC-043" (score de similaridade: 0.82)  
**Quando** o atendente envia: `"qual o custo de frete para 800 kg de carga perigosa para o Nordeste?"`  
**Então:**
- HTTP 200
- `escalationSignal.channel = "ramal 4500 — Gestão de Riscos"`
- `escalationSignal.reason` contém "PROC-043 está em revisão pelo Compliance"
- `answer` **não** contém nenhum valor calculado de frete
- `answer` **não** usa os multiplicadores do PROC-042v2-A
- `sourceDocument = null`
- `conflictPresentation = null`
- O `QueryLog` registra que o chunk `PROC-042v2-A` foi recuperado mas descartado por `BlockedTopic`

---

### VC-05 — Escalada por ausência de resposta no corpus, sem conhecimento geral

**Cenário:** pergunta sobre tema não coberto por nenhum chunk no corpus.

**Dado** que o índice não contém nenhum chunk sobre frete padrão (cargas abaixo de 500 kg)  
**Dado** que a busca retorna chunks com `confidenceScore` máximo de 0.43 (abaixo do threshold de 0.75)  
**Quando** o atendente envia: `"qual o prazo de entrega para uma carga de 200 kg de São Paulo para o Norte?"`  
**Então:**
- HTTP 200
- `escalationSignal.channel = "supervisor de atendimento"`
- `escalationSignal.reason` identifica o tema: "frete padrão para cargas abaixo de 500 kg não está coberto no corpus atual"
- `answer` **não** contém estimativas de prazo
- `answer` **não** menciona rotas, distâncias ou médias de entrega derivadas de conhecimento geral
- `sourceDocument = null`
- `confidenceScore` no `QueryLog` = 0.43 (registrado para auditoria)

---

### VC-06 — Distinção devolução/sinistro antes de responder

**Cenário:** pergunta ambígua que pode ser devolução (Contexto 2 → POL-001) ou sinistro (FAQ-38 → processo de sinistros). A distinção é obrigatória antes de qualquer orientação, conforme linguagem ubíqua.

**Dado** que o índice contém `POL-001-C` (procedimento de devolução) e `FAQ-38` (processo de sinistro, `documentClassification = informal`)  
**Quando** o atendente envia: `"carga chegou danificada — o que o cliente deve fazer?"`  
**Então:**
- HTTP 200
- `answer` contém pergunta de esclarecimento distinguindo devolução de sinistro
- `answer` **não** fornece um único fluxo sem distinção
- `sourceDocument = null` (aguardando esclarecimento)

**E quando** o atendente responde: `"a carga foi danificada durante o transporte, antes da entrega"`  
**Então:**
- `answer` orienta o processo de sinistro: registro em até 48h, canal sinistros@novatech.com.br
- `informalSourceWarning = true` (fonte é FAQ-38)
- `sourceDocument.name = "FAQ-Atendimento"`, `section = "item 38"`

**E quando** o atendente responde: `"o cliente recebeu a carga e quer devolver"`  
**Então:**
- `answer` orienta o processo de devolução: 7 dias úteis, Portal do Cliente, CT-e obrigatório
- `informalSourceWarning = false`
- `sourceDocument.name = "POL-001"`, `version = "3.1"`, `section = "seção 3.3"`

---

### VC-07 — Aviso de fonte informal presente e correto

**Cenário:** resposta gerada a partir de chunk do FAQ-Atendimento.

**Dado** que o índice contém `FAQ-15` com `documentClassification = informal` e o texto nega a existência do tier Platinum  
**Quando** o atendente envia: `"existe tier Platinum na NovaTech?"`  
**Então:**
- HTTP 200
- `informalSourceWarning = true`
- `answer` contém a informação: não existe tier Platinum; os tiers são Gold, Silver e Standard
- `answer` **não** contém o texto do aviso visual — o aviso é responsabilidade do Contexto 3 (renderizado via `informalSourceWarning = true`), não do Contexto 2
- `sourceDocument.name = "FAQ-Atendimento"`, `section = "item 15"`
- A resposta **não** é suprimida por ser de fonte informal — o campo `informalSourceWarning` sinaliza ao Contexto 3 que deve renderizar o aviso visual junto com a resposta

---

### VC-08 — Context budget respeitado com múltiplos chunks relevantes

**Cenário:** busca retorna mais chunks do que o budget permite — o sistema seleciona os top-5.

**Dado** que a busca retorna 9 chunks com `confidenceScore ≥ 0.75` para uma pergunta sobre frete e devolução combinados  
**Quando** o `prompt-builder.ts` constrói o prompt  
**Então:**
- Exatamente 5 chunks são incluídos no prompt (os 5 com maior `confidenceScore`)
- O total de tokens do slot de chunks ≤ 8.000 tokens
- O total de tokens do prompt completo ≤ 14.200 tokens (soma dos slots do ADR-0002)
- Os 4 chunks descartados não aparecem em nenhum `sourceDocument` da resposta
- O `QueryLog` registra todos os 9 chunks recuperados, com flag indicando quais foram incluídos no prompt

---

### VC-09 — Pergunta cruzando duas categorias (15% dos chamados)

**Cenário:** pergunta que requer chunks de duas fontes distintas — SLA e devolução. Dado do discovery: 15% das perguntas cruzam categorias.

**Dado** que o índice contém `SLA-2024-B` (SLAs para chamados gerais) e `POL-001-A` (prazo de devolução)  
**Quando** o atendente envia: `"cliente Gold com carga entregue ontem — qual o SLA se houver problema e qual o prazo para solicitar devolução?"`  
**Então:**
- HTTP 200
- `answer` responde as duas perguntas com suas fontes separadas
- `sourceDocument` é um array com dois elementos: `SLA-2024` (para SLA) e `POL-001` (para devolução)
- As informações de SLA **não** são atribuídas à POL-001 e vice-versa
- `confidenceScore` é calculado como o mínimo entre os dois scores (a resposta é tão confiável quanto seu chunk mais fraco)
- Latência ≤ 15 segundos (P95) mesmo com múltiplas fontes

---

### VC-10 — Schema da resposta válido e completo em qualquer cenário

**Cenário:** verificação de contrato — todo HTTP 200 deve retornar um JSON que passa na validação Zod.

**Dado** qualquer `Query` válida (coberta, não coberta, conflito, bloqueada)  
**Quando** o endpoint retorna HTTP 200  
**Então** o corpo deve satisfazer o schema:

```typescript
// src/shared/types.ts — fonte de verdade dos tipos

// Schema do REQUEST (contrato com o Contexto 3)
const QuerySchema = z.object({
  question:            z.string().min(1).max(2000),
  sessionId:           z.string().uuid().optional(),
  conversationHistory: z.array(z.object({
    role:    z.enum(["attendant", "assistant"]),
    content: z.string().min(1),
  })).max(6).optional(),  // 3 turnos × 2 mensagens = 6 mensagens máximo
});

// Schema da RESPONSE
const AssistantResponseSchema = z.object({
  queryId:               z.string().uuid(),
  answer:                z.string().min(1),
  sourceDocument:        z.union([SourceCitationSchema, z.array(SourceCitationSchema), z.null()]),
  confidenceScore:       z.number().min(0).max(1),
  escalationSignal:      z.union([EscalationSignalSchema, z.null()]),
  informalSourceWarning: z.boolean(),
  conflictPresentation:  z.union([ConflictPresentationSchema, z.null()]),
});

const SourceCitationSchema = z.object({
  name:    z.string().min(1),   // ex: "POL-001"
  version: z.string().min(1),   // ex: "3.1"
  section: z.string().min(1),   // ex: "seção 3.3"
  date:    z.string().min(1),   // ex: "15/01/2024"
});

const EscalationSignalSchema = z.object({
  channel: z.enum([
    "ramal 4500 — Gestão de Riscos",
    "Comercial",
    "sinistros@novatech.com.br",
    "supervisor de atendimento",
  ]),
  reason: z.string().min(1),
});

const ConflictPresentationSchema = z.object({
  versionA: z.object({
    value:  z.string(),
    source: SourceCitationSchema,
    informalSource: z.boolean(),  // true se este chunk é do FAQ informal
  }),
  versionB: z.object({
    value:  z.string(),
    source: SourceCitationSchema,
    informalSource: z.boolean(),
  }),
});
```

**Regras de consistência do schema (verificadas pelo `response-validator.ts`):**
- Se `escalationSignal ≠ null` e `conflictPresentation = null`: `sourceDocument` deve ser `null`
- Se `conflictPresentation ≠ null`: `sourceDocument` deve ser `null`
- `escalationSignal = null` e `sourceDocument = null` simultaneamente: inválido — retornar HTTP 500
- Se qualquer chunk dos `conflictPresentation.versionA` ou `versionB` for informal: `informalSourceWarning = true`

A ausência de qualquer campo obrigatório deve retornar HTTP 422 — testado com requests malformados no suite de contrato.

---

### VC-11 — HTTP 400 para input inválido

**Dado** que o endpoint recebe um request sem o campo `question`  
**Quando** o handler tenta processar  
**Então:**
- HTTP 400 com body `{ "error": "validation_error", "details": [...] }` no formato Zod
- Nenhuma chamada ao Azure AI Search ou Azure OpenAI é feita
- O erro é logado via `pino` com nível `warn`

---

### VC-12 — HTTP 500 com fallback seguro em falha de serviço externo

**Dado** que o Azure AI Search retorna erro 503 durante a busca  
**Quando** o handler processa uma `Query` válida  
**Então:**
- HTTP 500 com body `{ "error": "search_unavailable" }`
- O `QueryLog` registra a falha com timestamp e tipo de erro
- Nenhuma chamada ao Azure OpenAI é feita (fail fast)
- O erro é logado via `pino` com nível `error` e stack trace
- O atendente **não** recebe uma resposta parcial — recebe o erro claramente

---

### VC-13 — Esclarecimento ignorado: atendente envia nova pergunta sem responder

**Cenário:** o sistema pediu esclarecimento (ex: data do chamado para frete) e o atendente enviou uma pergunta diferente no turno seguinte.

**Dado** que no turno anterior o sistema solicitou a data do chamado para calcular frete especial  
**Quando** o atendente envia uma nova pergunta não relacionada: `"qual o SLA para cliente Silver?"`  
**Então:**
- HTTP 200
- O contexto de esclarecimento pendente é descartado — não é mencionado na resposta
- A nova pergunta é processada do zero, sem relação com o turno anterior
- `answer` responde sobre SLA Silver: resposta em até 4h úteis, resolução em até 48h úteis
- `sourceDocument.name = "SLA-2024"`, `version = "2024.1"`, `section = "seção 2"`
- O turno de esclarecimento descartado é registrado no `QueryLog` com flag `clarification_abandoned: true`

---

### VC-14 — Conflito entre chunk normativo e chunk informal

**Cenário:** dois chunks recuperados cobrem o mesmo tema — um normativo e um do FAQ informal — com valores divergentes. Caso concreto: FAQ item 45 (limiar de 10 fretes) vs PROC-042v2-D (limiar de 8 fretes).

**Dado** que o índice contém `PROC-042v2-D` (`documentClassification = normativo`, desconto a partir de 8 fretes/mês) e `FAQ-45` (`documentClassification = informal`, desconto a partir de 10 fretes/mês)  
**Quando** o atendente envia: `"a partir de quantos fretes especiais por mês o cliente tem desconto automático?"`  
**Então:**
- HTTP 200
- `conflictPresentation` não é null
- `conflictPresentation.versionA.value = "a partir de 8 fretes/mês (5% de desconto)"`
- `conflictPresentation.versionA.source.name = "PROC-042-v2"`, `version = "2.0"`, `section = "seção 4"`
- `conflictPresentation.versionA.informalSource = false`
- `conflictPresentation.versionB.value = "mais de 10 fretes/mês"`
- `conflictPresentation.versionB.source.name = "FAQ-Atendimento"`, `section = "item 45"`
- `conflictPresentation.versionB.informalSource = true`
- `informalSourceWarning = true` (pelo menos um dos chunks é informal)
- `sourceDocument = null`
- `answer` recomenda usar o documento normativo (PROC-042-v2) como fonte autoritativa e confirmar com o Comercial

---

## 6. Canais de escalada — valores válidos para `EscalationSignal.channel`

O enum `EscalationSignalSchema.channel` aceita apenas estes valores. Qualquer string fora desta lista é rejeitada pelo Zod:

| Canal | Quando usar |
|---|---|
| `"ramal 4500 — Gestão de Riscos"` | Frete de carga perigosa acima de 500 kg; devolução de carga perigosa; temas que dependem do PROC-043 (em revisão) |
| `"Comercial"` | Conflito documental sem resolução; frete reverso em contratos pré-01/12/2023; desconto de frete fora dos limiares automáticos; prazo expirado para devolução |
| `"sinistros@novatech.com.br"` | Sinistro — carga danificada em trânsito ou antes da entrega |
| `"supervisor de atendimento"` | Tema não coberto no corpus; pergunta fora do escopo do assistente; qualquer situação não mapeada nos canais acima |

---

## 7. Leituras obrigatórias antes de implementar

1. `docs/onboarding.md` — Bounded Contexts e Linguagem Ubíqua
2. `docs/adr/ADR-0001.md` — Decisão de modelo (GPT-4o)
3. `docs/adr/ADR-0002.md` — Estratégia de contexto e context budget
4. `docs/adr/ADR-0003.md` — Tratamento de documentos contraditórios
5. `docs/adr/ADR-0004.md` — Chunking especial para tabelas
6. `data/retrieval-corpus/chunks-novatech.md` — corpus de referência para testes
7. `prompts/system-prompt.md` — system prompt atual (v1)
8. `src/shared/types.ts` — tipos compartilhados entre contextos

> ⚠️ **Os arquivos ADR-0001 a ADR-0004 ainda não foram criados no repo.** O conteúdo de cada ADR está inline na seção 3 desta spec (Prior decisions) como referência temporária. Criar os arquivos em `docs/adr/` usando o template em `docs/adr/template.md` é tarefa do Tech Lead e deve ser concluída antes do início da implementação. Esta spec não pode ser considerada completa enquanto os ADRs não existirem como arquivos independentes.

---

*Próxima leitura: `specs/query-endpoint/plan.md` — decomposição técnica em tarefas*  
*v2.0 — ambiguidades do Tech Lead Review resolvidas — NovaTech × DB1*
