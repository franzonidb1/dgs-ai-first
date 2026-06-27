# Harness de Produto — Assistente NovaTech

> **Localização canônica no repo:** `docs/harness-produto.md`
> **Público:** Product Specialist, Tech Lead, QA, Delivery Manager, Gestor do Corpus
> **Versão:** 1.0
> **Distinção importante:** este é o harness de **produto** (governa a evolução do assistente ao longo do tempo). É distinto do harness de **runtime** (`docs/harness-governanca.md`), que governa cada resposta individual. O harness de runtime protege a resposta de hoje; o harness de produto protege a qualidade através das mudanças de amanhã.

---

## 1. O que este harness governa

O assistente vai mudar continuamente: documentos serão atualizados, o prompt será ajustado, novos guardrails serão adicionados. Cada mudança é uma oportunidade de melhorar — e um risco de quebrar o que já funcionava. O harness de produto é o sistema que garante três coisas:

1. **Processo de feedback** — como o feedback do atendente vira melhoria concreta
2. **Regression testing de produto** — como verificar que uma mudança não piora respostas existentes nem viola os guardrails do Cenário 2
3. **Human-in-the-loop de mudança** — quais mudanças exigem aprovação humana antes de ir a produção, e quem aprova

O princípio que atravessa os três: **nenhuma mudança vai a produção sem passar pela suite de regressão dos guardrails**. Os guardrails formalizados no Cenário 2 (DEVE / NÃO DEVE / QUANDO EM DÚVIDA) são invariantes que devem sobreviver a toda evolução — eles são a definição de "não piorou".

---

## 2. Processo de feedback — do reporte à melhoria

### 2.1 Entrada do feedback

O feedback chega por dois canais:

| Canal | Origem | Conteúdo |
|---|---|---|
| Reporte ativo | Atendente clica em "reportar" no card do Teams | Pergunta, resposta recebida, descrição do problema |
| Sinal passivo | QueryLog + métricas de qualidade | Respostas de baixa confiança, escaladas frequentes, casos HITL rejeitados |

Todo reporte ativo é automaticamente vinculado ao `QueryLog` da consulta original — preservando os chunks recuperados, os scores e a resposta gerada para investigação.

### 2.2 Triagem (Gestor do Corpus)

O Gestor do Corpus analisa cada reporte e identifica a causa-raiz. A pergunta-chave da triagem: **por que o assistente errou?**

| Causa-raiz | Sinal no QueryLog |
|---|---|
| Documento desatualizado no corpus | Chunk correto recuperado, mas conteúdo obsoleto |
| Documento ausente do corpus | Nenhum chunk relevante recuperado (score baixo) |
| Chunk errado recuperado | Score alto para chunk irrelevante (problema de chunking ou embedding) |
| Comportamento do modelo | Chunks corretos recuperados, mas resposta não os seguiu |
| Guardrail ausente ou insuficiente | Resposta violou uma regra que não estava codificada |

### 2.3 Classificação do tipo de mudança

A causa-raiz determina o tipo de mudança — e cada tipo tem um caminho diferente:

| Tipo de mudança | Quando | Ação | Responsável |
|---|---|---|---|
| **Documento** | Corpus desatualizado, ausente ou com chunk mal formado | Atualizar/adicionar documento → reindexação | Gestor do Corpus + área dona do documento |
| **Prompt** | Comportamento do modelo precisa de ajuste (tom, formato, instrução) | Ajustar `system-prompt.md` + registrar em `prompt-changelog.md` | Product Specialist |
| **Código** | Guardrail ausente ou harness insuficiente | Novo VC em `requirements.md` + implementação + novo guardrail | Tech Lead + Dev |

**Exemplos concretos dos casos reais:**

- A resposta sobre carga danificada (alucinação sobre tema sem documento formal) gera mudança de **documento** (criar a política de sinistros que não existe) **e** de **código** (invariante H1: confiança Alta exige fonte).
- A resposta sobre carga perigosa com FAQ gera mudança de **código** (guardrail G-N05 forçando escalada) — o FAQ não deve ser promovido a fonte normativa, então não é mudança de documento.
- A resposta que citou a seção errada da POL-001 gera mudança de **código** (validação de coerência afirmação/seção) ou de **prompt** (instrução de citação mais rigorosa).

### 2.4 O feedback fecha o ciclo

Toda mudança originada de feedback gera um artefato rastreável:

- Mudança de documento → entrada no log de reindexação com o reporte de origem
- Mudança de prompt → entrada no `prompt-changelog.md` com data, autor, motivo e resultado esperado
- Mudança de código → novo VC nos `requirements.md` usando o caso do reporte como fixture

O atendente que reportou recebe confirmação quando a correção entra em produção — fechando o loop e reforçando o uso do canal de feedback.

---

## 3. Regression testing de produto

### 3.1 O problema que resolve

Uma mudança que corrige um caso pode quebrar outro. Adicionar um documento pode introduzir um conflito. Ajustar o prompt para ser mais cauteloso em carga perigosa pode torná-lo cauteloso demais em temas simples. O regression testing de produto garante que **toda mudança é avaliada contra o conjunto inteiro de comportamentos esperados — não só contra o caso que motivou a mudança.**

### 3.2 As duas suites de regressão

Toda mudança — documento, prompt ou código — passa por duas suites antes de ir a produção:

**Suite 1 — Golden queries (qualidade das respostas)**

O conjunto `prompts/eval/golden-queries.json` contém perguntas de referência com respostas esperadas. A cada mudança, todas as golden queries são reexecutadas e comparadas com o baseline. Métricas monitoradas:

| Métrica | Critério de aprovação |
|---|---|
| Respostas corretas | Não pode cair em relação ao baseline |
| Respostas com fonte correta | 100% das não-escaladas |
| Escaladas indevidas (falso positivo) | Não pode aumentar |
| Respostas incorretas que viraram corretas | Ganho esperado da mudança |
| Latência P95 | Não pode ultrapassar 15s |

A regra: a mudança deve **melhorar ou manter** cada métrica. Se qualquer métrica piora, a mudança volta para ajuste (loop de retorno).

**Suite 2 — Guardrails do Cenário 2 (invariantes de comportamento)**

Esta é a suite que preserva os guardrails formalizados. Cada guardrail DEVE / NÃO DEVE / QUANDO EM DÚVIDA tem um teste de regressão correspondente. A suite verifica que toda mudança continua respeitando os guardrails:

| Categoria | Guardrail | Teste de regressão |
|---|---|---|
| DEVE | G-D01 — Citar fonte completa | Toda resposta não-escalada tem `source_document` com 4 campos |
| DEVE | G-D02 — Filtrar `doc_status` | Nenhum chunk histórico usado como fonte |
| DEVE | G-D04 — Escalada com canal específico | `channel` sempre no enum de 4 valores |
| DEVE | G-D05 — Verificar BlockedTopic primeiro | Tema bloqueado escala sem chamar o LLM |
| NÃO DEVE | G-N01 — Não aplicar regra geral sem exceção | Pergunta sobre devolução de carga perigosa → cita seção 3.2, nunca prazo de 7 dias |
| NÃO DEVE | G-N03 — Não inventar valor/prazo | Pergunta sobre tema ausente → escala, não estima |
| NÃO DEVE | G-N04 — Não escolher entre conflitantes | PROC-042 v1 vs v2 → apresenta ambos |
| NÃO DEVE | G-N05 — Não responder carga perigosa só com FAQ | Carga perigosa + chunk informal → escala ao ramal 4500 |
| NÃO DEVE | G-N07 — Não confundir devolução com sinistro | Carga danificada → pede esclarecimento |
| QUANDO EM DÚVIDA | G-F02 — Escalar carga perigosa em dúvida | Classificação incerta → escala |
| QUANDO EM DÚVIDA | G-F04 — Declarar ausência específica | Sem resposta → mensagem específica, não genérica |

**A regra inviolável:** uma mudança pode melhorar uma golden query, mas **nunca** pode fazer um teste de guardrail passar a falhar. Os guardrails são invariantes. Se uma mudança de prompt corrige um caso mas faz o assistente passar a escolher entre versões conflitantes (violando G-N04), a mudança é rejeitada — independentemente do ganho no caso original.

### 3.3 Os casos dos incidentes viram regressão permanente

Cada incidente e cada resposta incorreta de staging vira um teste de regressão permanente:

| Caso | Vira o teste de regressão |
|---|---|
| INC-01 (prazo de carga perigosa) | Pergunta de devolução de carga perigosa nunca retorna prazo de 7 dias |
| INC-02 (versão errada do PROC-042) | Resposta sobre frete nunca usa chunk com `doc_status = histórico` |
| INC-03 (falso negativo SLA Gold) | Pergunta sobre SLA Gold sempre encontra a resposta no corpus |
| Resposta 4 staging (carga danificada) | Confiança Alta nunca coexiste com fonte ausente |
| Resposta 6 staging (carga perigosa + FAQ) | Carga perigosa baseada só em FAQ sempre escala |

Isso garante que um erro corrigido nunca regride silenciosamente — a suite o pegaria.

---

## 4. Human-in-the-loop de mudança

### 4.1 O princípio

Nem toda mudança precisa do mesmo nível de aprovação. Atualizar um documento que corrige um erro factual é diferente de alterar um guardrail de segurança. O nível de aprovação é proporcional ao risco da mudança.

### 4.2 Matriz de aprovação por tipo e risco de mudança

| Tipo de mudança | Risco | Quem aprova | Regression obrigatória? |
|---|---|---|---|
| Reindexação de documento já validado (atualização de versão) | Baixo | Gestor do Corpus | Suite 1 + 2 |
| Adição de novo documento ao corpus | Médio | Gestor do Corpus + área dona do documento | Suite 1 + 2 |
| Ajuste de prompt que não toca guardrails | Médio | Product Specialist | Suite 1 + 2 |
| Ajuste de prompt que altera comportamento de guardrail | Alto | Product Specialist + Tech Lead | Suite 1 + 2 + revisão manual |
| Mudança em guardrail de enforcement via código | Alto | Tech Lead + Product Specialist | Suite 1 + 2 + revisão manual |
| Mudança que afeta tema sensível (carga perigosa, SLA contratual, valores de frete) | Crítico | Tech Lead + Product Specialist + validação do Compliance da NovaTech | Suite 1 + 2 + revisão manual + sign-off do cliente |
| Remoção de um guardrail | Crítico | Comitê (Tech Lead + Product Specialist + Delivery Manager) | Suite completa + justificativa documentada em ADR |

### 4.3 Regras de aprovação

**Nenhuma mudança pula a regressão.** Mesmo a mudança de menor risco (reindexação de documento já validado) passa pelas duas suites. O que varia é quem aprova e se há revisão manual além da automática.

**Mudança em tema sensível exige o cliente.** Qualquer alteração que afete carga perigosa, SLA contratual ou cálculo de valores de frete requer sign-off do Compliance da NovaTech — porque essas são áreas com consequência regulatória ou contratual. O assistente não decide sozinho, e a DB1 não decide sozinha, sobre esses temas.

**Remover um guardrail é a mudança de maior risco.** Adicionar guardrails é seguro — restringe comportamento. Remover um guardrail amplia o que o assistente pode fazer e precisa de justificativa documentada em ADR, aprovada por comitê. Um guardrail só é removido se a razão que o originou deixou de existir (ex: o PROC-043 foi publicado, então o tópico bloqueado de carga perigosa pode ser revisto).

### 4.4 O fluxo de promoção a produção

```
Mudança proposta
   → Regression (Suite 1 + Suite 2)
      → falhou? volta para ajuste (loop)
      → passou? segue para aprovação
   → Aprovação (conforme matriz de risco)
      → rejeitada? volta para ajuste
      → aprovada? promovida a produção
   → Confirmação ao atendente que reportou (se originada de feedback)
```

---

## 5. Métricas de qualidade monitoradas continuamente

O harness de produto monitora métricas que sinalizam quando uma mudança é necessária — alimentando o canal de sinal passivo do processo de feedback:

| Métrica | O que sinaliza | Limiar de atenção |
|---|---|---|
| Taxa de respostas incorretas (eval semanal) | Qualidade geral degradando | Acima da linha definida com a diretoria |
| Taxa de escaladas | Cobertura do corpus insuficiente ou threshold mal calibrado | Aumento sustentado mês a mês |
| Taxa de casos HITL rejeitados pelo supervisor | Assistente errando em temas sensíveis | Qualquer tendência de alta |
| Taxa de reportes de feedback por tema | Concentração de erros em um tema específico | Pico em um tema = gap de corpus |
| Latência P95 | Degradação de performance | Acima de 15s |
| Cobertura de teste por módulo | Erosão da rede de proteção | Abaixo de 80% |

Métricas que cruzam o limiar de atenção geram automaticamente um item de investigação para o Gestor do Corpus — o sinal passivo que complementa o reporte ativo do atendente.

---

## 6. Como os três componentes se conectam

O harness de produto é um ciclo, não três processos isolados:

1. O **feedback** (ativo do atendente ou passivo das métricas) identifica que algo precisa mudar.
2. A **triagem** classifica o tipo de mudança e direciona para o caminho certo (documento, prompt ou código).
3. O **regression testing** garante que a mudança melhora sem quebrar — validando contra golden queries e contra os guardrails do Cenário 2.
4. O **human-in-the-loop** aprova conforme o risco — do Gestor do Corpus sozinho até o comitê com sign-off do cliente.
5. A mudança vai a **produção** e o ciclo recomeça, com o novo caso virando regressão permanente.

O elemento que mantém a integridade do sistema ao longo do tempo é a **suite de guardrails como invariante**. Os guardrails do Cenário 2 não são sugestões que podem ser relaxadas para fazer uma mudança passar — são a definição de comportamento aceitável que toda evolução deve preservar. Uma mudança que melhora uma métrica mas viola um guardrail não é uma melhoria; é uma regressão de comportamento disfarçada de ganho de qualidade.

---

*Harness de produto — NovaTech × DB1*
*Distinto de: `docs/harness-governanca.md` (harness de runtime)*
*Preserva: os guardrails formalizados em `prompts/guardrails.md` (Cenário 2)*
