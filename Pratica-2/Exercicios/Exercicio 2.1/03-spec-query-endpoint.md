# Spec — Query Endpoint (Módulo Principal)

> **Localização no repo:** `specs/query-endpoint/requirements.md`  
> **Formato:** Spec Driven Development (SDD)  
> **Módulo:** `src/functions/query/` + `src/services/`  
> **Responsável pela spec:** Product Specialist  
> **Responsável pela implementação:** Dev Sênior (referenciado no AGENTS.md como Dev 2.2)

---

## 1. Outcomes — o que o sucesso parece

O query endpoint existe para que um atendente da NovaTech consiga obter, em menos de 30 segundos, uma resposta fundamentada na documentação oficial para perguntas sobre SLAs, regras de frete e política de devolução — sem precisar abrir SharePoint, Confluence ou pasta de rede.

**O endpoint entrega sucesso quando:**

1. O atendente recebe uma resposta com citação de fonte que permite verificar a informação no documento original sem esforço adicional.
2. Quando há conflito documental (ex: PROC-042 v1 e v2), o atendente vê ambas as versões com suas fontes — nunca uma escolha arbitrária do modelo.
3. Quando o assistente não tem resposta segura, o atendente sabe exatamente para onde escalar — nunca fica sem direção.
4. O assistente nunca usa conhecimento geral do LLM como substituto do corpus — toda informação é rastreável a um chunk indexado.

**Métricas de sucesso (verificáveis no go-live):**
- Latência P95 ≤ 15 segundos da pergunta à resposta completa
- Taxa de respostas com `sourceDocument` preenchido: 100% das respostas não-escaladas
- Taxa de escalada orientada (com canal específico): 100% dos casos sem resposta no corpus
- Taxa de respostas baseadas em conhecimento geral do LLM: 0%

---

## 2. Scope boundaries — o que está dentro e fora desta spec

### Dentro do escopo

- Recebimento e validação da pergunta do atendente via HTTP POST
- Busca de chunks relevantes no Azure AI Search
- Verificação da lista de tópicos bloqueados
- Detecção de conflito entre chunks recuperados
- Construção do prompt dentro do context budget (ADR-0002)
- Chamada ao Azure OpenAI GPT-4o (ADR-0001)
- Validação da resposta gerada
- Retorno da resposta estruturada com `sourceDocument`, `confidenceScore` e `escalationSignal` quando aplicável

### Fora do escopo desta spec

- Renderização do card no Teams — coberta em `specs/teams-bot/`
- Persistência do log de consultas — coberta em `specs/feedback-api/`
- Indexação de documentos no Azure AI Search — coberta em `specs/pipeline-ingestao/`
- Autenticação e autorização do atendente — coberta em `specs/query-endpoint/plan.md` (decisão de infraestrutura)

---

## 3. Constraints — restrições que a implementação não pode violar

### Restrições derivadas das ADRs do Cenário 1

| Restrição | Origem | Impacto na implementação |
|---|---|---|
| Modelo: Azure OpenAI GPT-4o | ADR-0001 | Nenhum outro modelo pode ser usado em produção |
| Context budget: ~4K system prompt + ~8K chunks + pergunta + 3 turnos | ADR-0002 | O `prompt-builder.ts` deve garantir que o total nunca exceda a janela do modelo |
| Máximo de 5 chunks por consulta (~1.500 tokens cada) | ADR-0002 | O `search.ts` retorna no máximo 5 chunks mesmo que o índice tenha mais resultados relevantes |
| Documentos contraditórios: metadado de vigência + instrução no prompt | ADR-0003 | O sistema não escolhe automaticamente entre versões — apresenta ambas |
| Chunking de tabelas requer atenção especial | ADR-0004 | O `search.ts` deve recuperar o chunk de tabela completa quando o chunk adjacente for insuficiente |

### Restrições de produto (derivadas da spec de RAG do Cenário 1)

- O campo `sourceDocument` é obrigatório em toda resposta não-escalada. Respostas sem fonte são proibidas.
- O assistente nunca usa conhecimento geral do modelo como fallback. Se `confidenceScore < 0.75` e não há chunks suficientes, a resposta é uma escalada orientada.
- A lista de tópicos bloqueados é verificada **antes** de usar os chunks recuperados — não depois.
- Fontes com `documentClassification = informal` geram `informalSourceWarning = true` na resposta, sem suprimir o conteúdo.

### Restrições de stack (Anexo D)

- Linguagem: TypeScript (ESM, `strict: true`, target ES2022)
- Validação de input/output: Zod (obrigatório — sem validação manual de tipos)
- Logging: `pino` (sem `console.log` em nenhum ponto do código de produção)
- Padrão de função: Azure Functions v4 (handler exportado como função nomeada)
- Cobertura de testes: ≥ 80% de linhas (configurado no `vitest.config.ts`)

---

## 4. Prior decisions — decisões já tomadas que esta spec não reabre

| Decisão | ADR | O que está fechado |
|---|---|---|
| Modelo GPT-4o via Azure OpenAI | ADR-0001 | Escolha do modelo, integração com ecossistema Microsoft |
| Context budget e estratégia de janela | ADR-0002 | Distribuição de tokens entre system prompt, chunks e histórico |
| Tratamento de documentos contraditórios via metadado + prompt | ADR-0003 | Estratégia: metadado de vigência no pipeline + instrução explícita no system prompt |
| Chunking especial para tabelas | ADR-0004 | Problema identificado no protótipo open-source; solução a implementar no Contexto 1 |
| Azure AI Search como índice vetorial | Cenário 2 (cenário narrativo) | Substituição do ChromaDB do protótipo |
| Bot Framework para integração com Teams | Cenário 2 | Canal de entrega principal |

**Esta spec não discute nem reabre nenhuma dessas decisões.** Se surgirem dúvidas sobre qualquer uma delas durante a implementação, criar um novo ADR em `docs/adr/` e submeter para revisão do Tech Lead — não alterar silenciosamente o comportamento.

---

## 5. Verification criteria — critérios de aceitação por comportamento

Cada critério é formulado como um caso de teste verificável pelo QA. O formato é: **Dado [contexto] → Quando [ação] → Então [resultado esperado].**

---

### VC-01 — Resposta com fonte para pergunta coberta pelo corpus

**Dado** que o corpus contém o chunk `SLA-2024-C` (SLAs para incidentes críticos) com `documentStatus = vigente`  
**Quando** o atendente pergunta "qual o prazo de resposta para cliente Gold em incidente crítico?"  
**Então** a resposta deve:
- Conter o valor correto: "30 minutos"
- Ter `sourceDocument.name = "SLA-2024"`, `sourceDocument.version = "2024.1"`, `sourceDocument.section = "seção 2"`, `sourceDocument.date = "02/01/2024"`
- Ter `confidenceScore ≥ 0.75`
- Ter `escalationSignal = null`
- Ser retornada em ≤ 15 segundos (P95)

---

### VC-02 — Apresentação de conflito documental sem escolha

**Dado** que o corpus contém `PROC-042-A` (fator de peso v1: 1.2) e `PROC-042v2-A` (fator de peso v2: 1.15) ambos com `documentStatus = vigente_transitório`  
**Quando** o atendente pergunta "qual o fator de peso para uma carga de 2.000 kg?" sem informar a data do chamado  
**Então** a resposta deve:
- Solicitar a data do chamado (campo obrigatório para frete, conforme linguagem ubíqua)
- **Não** retornar um valor calculado antes de receber a data
- Após a data ser informada: usar o documento correspondente ao período (v1 para chamados anteriores a 01/12/2023, v2 para chamados a partir de 01/12/2023)
- Ter `sourceDocument` preenchido com a versão usada
- **Não** apresentar os dois valores como igualmente válidos para a mesma data

---

### VC-03 — Escalada orientada com canal específico para tópico bloqueado

**Dado** que "frete de carga perigosa acima de 500 kg" está na lista de tópicos bloqueados (PROC-043 em revisão)  
**Dado** que o corpus contém o chunk `PROC-042v2-A` com menção a "cargas perigosas seguem PROC-043"  
**Quando** o atendente pergunta "qual o custo de frete para 800 kg de carga perigosa para o Norte?"  
**Então** a resposta deve:
- Ter `escalationSignal.channel = "ramal 4500 — Gestão de Riscos"`
- Ter `escalationSignal.reason = "PROC-043 está em revisão pelo Compliance e não está disponível"`
- **Não** calcular nem estimar um valor de frete — nem com os multiplicadores do PROC-042v2
- **Não** usar o chunk `PROC-042v2-A` como base para uma resposta parcial

---

### VC-04 — Ausência de resposta declarada, nunca substituída por conhecimento geral

**Dado** que o corpus não contém nenhum chunk sobre frete padrão (cargas abaixo de 500 kg)  
**Quando** o atendente pergunta "qual o prazo de entrega para uma carga de 200 kg de São Paulo para o Norte?"  
**Então** a resposta deve:
- Ter `escalationSignal` preenchido com canal "supervisor de atendimento"
- Ter `escalationSignal.reason` identificando que o tema não está coberto no corpus
- **Não** estimar um prazo baseado em conhecimento geral sobre rotas de transporte
- **Não** ter `sourceDocument` preenchido (ausência de fonte é esperada neste caso)
- O texto da resposta deve mencionar o tema identificado: "frete padrão para cargas abaixo de 500 kg"

---

### VC-05 — Aviso de fonte informal para respostas baseadas no FAQ

**Dado** que o corpus contém o chunk `FAQ-15` (tier Platinum inexistente) com `documentClassification = informal`  
**Quando** o atendente pergunta "o cliente diz que é Platinum — esse tier existe?"  
**Então** a resposta deve:
- Ter `informalSourceWarning = true`
- O texto da resposta deve conter o aviso: "⚠️ Fonte não validada por Compliance"
- Conter a informação correta: não existe tier Platinum; os tiers são Gold, Silver e Standard
- **Não** suprimir a resposta por ser de fonte informal — exibir com aviso

---

### VC-06 — Não ultrapassar o context budget

**Dado** que a busca retorna 8 chunks relevantes com scores acima de 0.75  
**Quando** o `prompt-builder.ts` constrói o prompt  
**Então**:
- Apenas os 5 chunks com maior score devem ser incluídos no prompt
- O total de tokens do prompt (system prompt + chunks + pergunta + histórico) deve ser ≤ 128K tokens (limite do GPT-4o)
- O total de tokens alocados para chunks deve ser ≤ ~8K tokens
- Os 3 chunks descartados não devem aparecer na resposta nem influenciar a citação de fonte

---

### VC-07 — Pergunta que cruza duas categorias

**Dado** que 15% das perguntas cruzam categorias (dado do discovery)  
**Quando** o atendente pergunta "cliente Gold com frete especial de 1.500 kg — qual o SLA se houver problema na entrega e qual o prazo para ele solicitar devolução?"  
**Então** a resposta deve:
- Recuperar chunks de SLA-2024 (para o SLA de incidente) e POL-001 (para o prazo de devolução)
- Citar ambas as fontes na resposta, cada informação com sua fonte correspondente
- **Não** misturar as fontes — SLA vem do SLA-2024, devolução vem da POL-001
- Ter dois `sourceDocument` na resposta (ou estrutura equivalente para múltiplas fontes)
- Ser retornada em ≤ 15 segundos (P95) mesmo com múltiplas fontes

---

### VC-08 — Latência sob carga normal

**Dado** cenário de operação normal com 45 consultas simultâneas  
**Quando** 45 atendentes fazem perguntas ao mesmo tempo  
**Então**:
- P95 de latência ≤ 15 segundos
- P99 de latência ≤ 30 segundos
- Nenhuma consulta deve retornar erro 5xx
- O context budget de cada consulta deve ser respeitado independentemente das outras

---

### VC-09 — Distinção automática entre devolução e sinistro

**Dado** que devolução e sinistro são processos distintos com canais diferentes (conforme linguagem ubíqua)  
**Quando** o atendente pergunta "carga chegou danificada — o que faço?"  
**Então** a resposta deve:
- Solicitar esclarecimento: "A carga foi danificada durante o transporte (antes ou durante a entrega) ou o cliente quer devolver após ter recebido em bom estado?"
- **Não** responder com um único fluxo sem distinção
- Após o esclarecimento: se sinistro → canal sinistros@novatech.com.br (48h para registro); se devolução → POL-001 v3.1 (7 dias úteis, Portal do Cliente)

---

### VC-10 — Retorno estruturado verificável pelo QA

**Dado** qualquer pergunta válida  
**Quando** o endpoint retorna HTTP 200  
**Então** o corpo da resposta deve ser um objeto JSON que passa na validação do schema Zod com os campos:
```typescript
{
  answer: string,                    // texto da resposta em linguagem natural
  sourceDocument: SourceCitation | SourceCitation[] | null,  // null apenas em escaladas
  confidenceScore: number,           // 0–1; < 0.75 implica escalada
  escalationSignal: EscalationSignal | null,
  informalSourceWarning: boolean,
  conflictPresentation: ConflictPresentation | null,
  queryId: string                    // UUID para vinculação ao QueryLog
}
```
A ausência de qualquer campo obrigatório deve retornar HTTP 422 no teste de contrato.

---

## 6. Fluxo de implementação recomendado (SDD)

O Tech Lead e os Devs devem usar este fluxo ao implementar o módulo:

1. **`specs/query-endpoint/requirements.md`** (este arquivo) — ler antes de escrever código
2. **`specs/query-endpoint/plan.md`** — decomposição técnica em tarefas por arquivo
3. **`specs/query-endpoint/tasks.md`** — checklist de tarefas com critério de "done" por tarefa
4. Implementar na ordem: schema Zod → `search.ts` → `prompt-builder.ts` → `completion.ts` → `response-validator.ts` → `handler.ts`
5. Testes unitários para cada serviço antes de integrar no handler
6. Testes de integração usando o corpus simulado em `data/retrieval-corpus/chunks-novatech.md`
7. Testes contra os golden queries em `prompts/eval/golden-queries.json`

---

*Próxima leitura: `specs/query-endpoint/plan.md` — decomposição técnica*  
*Gerado na fase de estruturação — NovaTech × DB1*
