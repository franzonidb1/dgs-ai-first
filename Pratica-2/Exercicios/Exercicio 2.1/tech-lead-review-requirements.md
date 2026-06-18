# Tech Lead Review — `specs/query-endpoint/requirements.md`

> **Tipo:** Revisão crítica de ambiguidades e gaps  
> **Documento revisado:** `specs/query-endpoint/requirements.md`  
> **Perspectiva:** Tech Lead — quem precisa implementar sem perguntar nada ao Product Specialist  
> **Resultado:** 14 pontos em 3 categorias de severidade

---

## Resumo executivo

| Categoria | Quantidade | Impacto |
|---|---|---|
| 🔴 Bloqueantes | 5 | Implementação ambígua ou impossível — precisam de resposta antes de qualquer código |
| 🟡 Comportamentais | 6 | Comportamentos não determinísticos em produção — precisam de decisão antes do go-live |
| 🔵 Estruturais | 3 | Gaps de contrato entre módulos — criam inconsistência na integração entre contextos |

**Recomendação:** nenhuma implementação deve começar antes que os 5 bloqueantes sejam respondidos pelo Product Specialist e os 3 estruturais sejam resolvidos no contrato de tipos (`src/shared/types.ts`).

---

## 🔴 Bloqueantes — implementação impossível sem resposta

### B-01 — O que é exatamente um "turno" no histórico de 3 turnos?

**Onde aparece:** ADR-0002, VC-02, VC-06

**Problema:** ADR-0002 diz "histórico limitado a 3 turnos". Os VCs de esclarecimento (VC-02 e VC-06) definem fluxos de 2 turnos para resolver uma única pergunta. Um "turno" é um par pergunta+resposta, ou uma mensagem individual? Se os dois turnos de esclarecimento contam para o limite, a terceira pergunta real do atendente já estoura o histórico — e o prompt-builder.ts descarta o contexto da conversa que ainda está em andamento.

**O que precisa ser definido:**
- Definição formal de "turno": par (pergunta do atendente + resposta do assistente) ou mensagem individual.
- Se turnos de esclarecimento contam para o limite de 3 ou são tratados como overhead transparente do mesmo turno lógico.

---

### B-02 — Como o sistema detecta "conflito" nos chunks recuperados?

**Onde aparece:** C-P04, ADR-0003, VC-03, `response-validator.ts`

**Problema:** C-P04 diz que o `response-validator.ts` "deve rejeitar respostas que resolvam o conflito silenciosamente". ADR-0003 diz que o sistema "detecta conflito". Mas o algoritmo de detecção nunca está definido. O sistema compara os chunks semanticamente? Verifica se dois chunks têm o mesmo `doc_codigo` com versões diferentes? Usa uma lista estática de `ConflictRecord` vinda do Contexto 1? Cada abordagem tem falsos positivos diferentes e implica código completamente distinto no `response-validator.ts`.

**O que precisa ser definido:**
- O mecanismo de detecção: (a) lista estática de pares conflitantes produzida pelo Contexto 1, (b) heurística por `doc_codigo` idêntico + versões diferentes, ou (c) ambos em ordem de precedência.
- Sem isso, o Dev implementa qualquer heurística que pareça razoável — e o comportamento em produção será imprevisível.

---

### B-03 — Como a lista de `BlockedTopic` chega ao Contexto 2 em runtime?

**Onde aparece:** C-P03, ADR-0002 (system prompt), Scope boundaries

**Problema:** Há uma contradição direta entre dois trechos da spec:
- ADR-0002 diz que o system prompt contém "a lista de `BlockedTopic` atual".
- ADR-0002 também diz que "o system prompt é fixo por versão — não é gerado dinamicamente em runtime".

Se a lista de `BlockedTopic` muda quando um novo documento é indexado (ex: quando o PROC-043 for publicado, ele sai da lista), o system prompt não pode ser fixo e conter a lista atualizada ao mesmo tempo. Isso bloqueia diretamente a implementação do `prompt-builder.ts`.

**O que precisa ser decidido (duas opções mutuamente exclusivas):**
- **Opção A:** a lista é injetada dinamicamente no prompt a cada request — então o system prompt não é totalmente fixo e o ADR-0002 precisa ser atualizado.
- **Opção B:** a lista é verificada via lookup antes da busca de chunks, fora do prompt — então o ADR-0002 está errado ao colocá-la no system prompt.

---

### B-04 — `confidenceScore` de múltiplos chunks: qual valor reportar?

**Onde aparece:** VC-09, VC-10 (schema), C-P02

**Problema:** VC-09 define que para perguntas que cruzam categorias, o `confidenceScore` da resposta é "o mínimo entre os dois scores". Mas o critério "mínimo" é arbitrário e cria um problema operacional não resolvido: se um chunk tem score 0.9 e outro 0.4 (abaixo do threshold de 0.75), o conjunto total escala ou responde? O "mínimo" faria escalar — mas o chunk com score 0.9 tem resposta válida para metade da pergunta. Além disso, o critério não está justificado — por que não a média?

**O que precisa ser definido:**
- A função de agregação de scores para respostas multi-chunk (mínimo, média, ponderada, ou por chunk independente).
- Se o threshold de 0.75 é aplicado por chunk individualmente (cada chunk precisa passar) ou ao score agregado.
- O comportamento quando parte da pergunta tem chunk válido e parte não tem: responde o que sabe + escala o resto, ou escala tudo?

---

### B-05 — O `QueryLog` é criado pelo Contexto 2 ou pelo Contexto 4?

**Onde aparece:** Scope boundaries, VC-04, VC-12, Contexto 4 (bounded contexts)

**Problema:** A spec diz que o Contexto 2 "emite o `QueryLog` para consumo pelo Contexto 4". Mas VC-04 diz "O `QueryLog` registra que o chunk foi descartado por `BlockedTopic`" — uma informação que só existe no Contexto 2. VC-12 diz "O `QueryLog` registra a falha" em caso de erro do Azure AI Search — que também ocorre no Contexto 2, antes de qualquer comunicação com o Contexto 4. Se o Contexto 4 cria o `QueryLog`, como ele sabe dos chunks descartados e das falhas intermediárias do Contexto 2?

**O que precisa ser definido:**
- Quem cria o objeto `QueryLog`: o handler do Contexto 2 (e emite via evento/queue) ou o Contexto 4 (ao receber o evento).
- O mecanismo de emissão: event/queue assíncrono, chamada HTTP fire-and-forget, ou escrita direta no storage.
- O que acontece com o `QueryLog` quando o Contexto 4 está indisponível — o Contexto 2 falha ou continua?

---

## 🟡 Comportamentais — não determinísticos em produção

### C-01 — Quais "termos correlatos" disparam o fluxo de esclarecimento devolução/sinistro?

**Onde aparece:** C-P06, VC-06

**Problema:** C-P06 diz "carga danificada ou termos correlatos, conforme linguagem ubíqua". Mas a linguagem ubíqua (`docs/onboarding.md`) define os termos do domínio — não define uma lista de gatilhos para o fluxo de esclarecimento. "Avaria", "produto com defeito", "embalagem violada", "lacre rompido", "carga molhada" — cada um dispara o esclarecimento? O LLM decide baseado em semântica? Com que critério? O VC-06 testa apenas uma formulação exata ("carga chegou danificada") — não cobre as variações.

**Resolução sugerida:** Adicionar à spec uma lista explícita de intenções ou expressões que disparam o esclarecimento. Ou definir que o gatilho é semântico (o LLM classifica a intenção antes de buscar chunks) e documentar como isso é testado além do VC-06.

---

### C-02 — O que acontece se o atendente fizer uma segunda pergunta sem responder ao esclarecimento?

**Onde aparece:** VC-02, VC-06, ADR-0002 (histórico)

**Problema:** VC-02 e VC-06 definem fluxos onde o sistema aguarda a resposta do atendente ao pedido de esclarecimento. Mas a spec não define o que acontece se o atendente ignorar a pergunta de esclarecimento e enviar uma nova pergunta completamente diferente. O sistema insiste no esclarecimento pendente? Descarta o contexto e processa a nova pergunta? Isso interage diretamente com o limite de 3 turnos do histórico (B-01) e com a forma como o `conversationHistory` é mantido entre requests.

**Resolução sugerida:** Definir política de descarte de esclarecimento pendente: após 1 turno sem resposta, descartar o contexto pendente e processar a nova pergunta do zero. Adicionar VC específico para esse caso.

---

### C-03 — Qual o comportamento quando a resposta tem conflito E fonte informal?

**Onde aparece:** C-P04, C-P05, VC-03, VC-07

**Problema:** A spec define bem os casos isolados: conflito → `ConflictPresentation`; fonte informal → `informalSourceWarning = true`. Mas não define a combinação. O FAQ-Atendimento item 8 tem informação sobre frete especial — se esse chunk for recuperado junto com um chunk do PROC-042 normativo, temos conflito com fonte informal. O schema do VC-03 mostra `sourceDocument = null` em conflitos, mas não define o valor de `informalSourceWarning` nesse caso. Os dois campos coexistem? Como?

**Resolução sugerida:** Adicionar um VC específico: "conflito entre chunk normativo e chunk informal". Definir explicitamente: `conflictPresentation` preenchida com ambas as versões, `informalSourceWarning = true` se qualquer um dos chunks for informal, `sourceDocument = null`. Adicionar campo `informalSourceWarning` ao `ConflictPresentationSchema`.

---

### C-04 — Como detectar "tabela parcial" num chunk? O critério é subjetivo.

**Onde aparece:** ADR-0004, `response-validator.ts`

**Problema:** ADR-0004 diz que tabela parcial é "detectável por ausência de header ou de pelo menos uma linha de dados". Mas um chunk pode ter header e apenas 1 linha de 5 — está completo ou parcial? E se começa no meio da tabela mas tem o footer? O critério gera falsos negativos e coloca lógica de ingestão no Contexto 2 — violando os bounded contexts: a completude de uma tabela é informação produzida no Contexto 1 (chunking), não detectável pelo Contexto 2 em runtime.

**Resolução sugerida:** O critério de "tabela completa" deve vir do metadado do chunk, produzido pelo Contexto 1 durante o chunking. Adicionar campo `chunk_is_complete_table: boolean` ao metadado do `Chunk` e remover a lógica de detecção heurística do `response-validator.ts`.

---

### C-05 — O threshold 0.75 é hard-coded ou configurável?

**Onde aparece:** C-P02, VC-05, Seção 1 — Outcomes

**Problema:** O threshold de 0.75 aparece como valor fixo em C-P02 e VC-05. Mas a spec de RAG do Cenário 1 dizia explicitamente que esse é um "valor inicial — deve ser calibrado com base nos primeiros 30 dias de operação em produção". A spec atual não menciona esse processo de calibração nem onde o valor é configurado. Se está hardcoded, como o time ajusta após go-live sem fazer um novo deploy?

**Resolução sugerida:** Definir que o threshold é configurável via variável de ambiente (`CONFIDENCE_THRESHOLD`, default `0.75`). Documentar nos outcomes que o threshold deve ser revisado após 30 dias com base nos dados do `QueryLog`. Adicionar constraint de stack para esse padrão de configuração.

---

### C-06 — A métrica "0% de respostas baseadas em conhecimento geral" não é testável automaticamente.

**Onde aparece:** Seção 1 — Outcomes (métricas)

**Problema:** A métrica existe — mas não há nenhum mecanismo definido na spec para verificá-la automaticamente. O `response-validator.ts` valida o schema da resposta, não se o conteúdo do campo `answer` está contido nos chunks recuperados. Essa verificação só pode ser feita manualmente, com os golden queries de `prompts/eval/`. Incluir como métrica de go-live cria uma expectativa que o time não consegue monitorar em produção.

**Resolução sugerida:** Substituir por métricas automaticamente verificáveis: "100% das respostas não-escaladas têm `sourceDocument` preenchido" (já existe) e "0% das respostas têm `sourceDocument = null` com `escalationSignal = null`" (nova). Mover "0% de respostas baseadas em conhecimento geral" para o processo de eval manual com frequência definida (semanal nos primeiros 30 dias).

---

## 🔵 Estruturais — gaps de contrato entre módulos

### E-01 — O schema do request (`Query`) não está definido — só o da response.

**Onde aparece:** VC-10, VC-11, C-S02

**Problema:** VC-10 define o schema completo da `AssistantResponse` em Zod. VC-11 menciona "request sem o campo `question`" — implicando que `question` existe — mas o schema do request nunca é definido formalmente. Quais são os campos obrigatórios? Existe `sessionId`? `attendantId`? `conversationHistory`? Sem esse schema, o Dev do Contexto 3 (que constrói o request) e o Dev do Contexto 2 (que o valida) vão implementar contratos diferentes.

**Resolução sugerida:** Adicionar à seção 5 (ou ao VC-10) o schema Zod do request:

```typescript
const QuerySchema = z.object({
  question:            z.string().min(1).max(2000),
  sessionId:           z.string().uuid().optional(),
  conversationHistory: z.array(z.object({
    role:    z.enum(["attendant", "assistant"]),
    content: z.string().min(1),
  })).max(6).optional(),  // 3 turnos × 2 mensagens cada
});
```

---

### E-02 — O `answer` contém o aviso de fonte informal ou apenas o campo `informalSourceWarning`?

**Onde aparece:** VC-07, C-P05, Scope boundaries (Contexto 3)

**Problema:** VC-07 diz que o `answer` deve conter o texto "⚠️ Fonte não validada por Compliance". C-P05 diz que `informalSourceWarning = true` é o campo que o Contexto 3 usa para exibir o aviso visual. Se o aviso já está no `answer` (Contexto 2) e o Contexto 3 também renderiza o campo `informalSourceWarning`, o atendente vê o aviso duplicado. Se o aviso não está no `answer`, o VC-07 está errado. É uma contradição direta que vai gerar inconsistência entre os dois Devs que implementarem cada contexto.

**Resolução sugerida:** Definir explicitamente — e este é o padrão correto de separação de responsabilidades:
- O `answer` contém apenas o conteúdo da resposta em linguagem natural, sem marcações de UI.
- `informalSourceWarning = true` é o sinal para o Contexto 3 renderizar o aviso visual onde e como for adequado à interface.
- Corrigir o VC-07: remover o texto do aviso do `answer` e verificar apenas que `informalSourceWarning = true`.

---

### E-03 — As ADRs referenciadas na seção 7 não existem no repo.

**Onde aparece:** Seção 7 — Leituras obrigatórias, Seção 3 — Prior decisions

**Problema:** A seção 7 lista como leitura obrigatória `docs/adr/ADR-0001.md`, `ADR-0002.md`, `ADR-0003.md` e `ADR-0004.md`. O repo (Anexo D) contém apenas `docs/adr/template.md` — os ADRs não foram criados. Qualquer agente de IA ou Dev novo que seguir a seção 7 literalmente vai encontrar 404s. A spec está referenciando documentos que são pré-requisitos dela mesma como se já existissem.

**Resolução sugerida:** Criar os 4 ADRs em `docs/adr/` antes de marcar esta spec como pronta. Os ADRs devem ser o primeiro artefato do exercício — esta spec depende deles, não o contrário. Enquanto não existem, atualizar a seção 7 com:

```
> ⚠️ Os arquivos ADR-0001 a ADR-0004 ainda não foram criados no repo.
> O conteúdo de cada ADR está inline na seção 3 desta spec (Prior decisions)
> como referência temporária. Criar os arquivos em docs/adr/ é tarefa do Tech Lead
> antes do início da implementação.
```

---

## Próximos passos recomendados

**Antes de qualquer implementação:**

1. Product Specialist responde os 5 bloqueantes (B-01 a B-05) e atualiza a spec.
2. Tech Lead cria os 4 ADRs em `docs/adr/` usando o template existente.
3. `src/shared/types.ts` é atualizado com o schema do request (E-01) e as correções do contrato `informalSourceWarning` (E-02).

**Antes do go-live:**

4. Processo de calibração do threshold (C-05) documentado nos ADRs de operação.
5. Processo de eval manual para C-06 definido em `prompts/eval/`.
6. VCs adicionados para os casos C-02 (esclarecimento ignorado) e C-03 (conflito + fonte informal).

---

*Revisão produzida na fase de estruturação — NovaTech × DB1*  
*Documento revisado: `specs/query-endpoint/requirements.md`*  
*Próximo passo: atualizar requirements.md com as resoluções e criar os ADRs*
