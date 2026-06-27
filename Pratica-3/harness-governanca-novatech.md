# Harness de Governança — Assistente NovaTech

> **Localização canônica no repo:** `docs/harness-governanca.md`
> **Público:** Tech Lead, Product Specialist, QA, Delivery Manager + agentes de IA
> **Versão:** 1.0
> **Base:** ADRs (Cenário 1) + AGENTS.md, specs SDD, guardrails (Cenário 2) + incidentes de desenvolvimento
> **Propósito:** amarrar structured outputs, validação determinística e human-in-the-loop num sistema único de governança que torna o assistente confiável antes do go-live

---

## 1. Por que o harness existe

Os testes internos revelaram que **12% das respostas estavam incorretas** — alucinação, documento desatualizado e chunk incorreto recuperado. Além disso, três fragilidades estruturais foram identificadas:

| Fragilidade | Causa | O harness resolve com |
|---|---|---|
| Respostas em texto livre sem campos obrigatórios garantidos | O modelo às vezes "esquece" de incluir a fonte e nada impede a resposta de seguir | Structured outputs + validação determinística (camadas 2 e 3) |
| Código gerado por IA que ignora regras do AGENTS.md | Dev gerou módulo de feedback sem Zod e com log de dados sensíveis | Revisão crítica obrigatória de código gerado por IA (seção 6) |
| Respostas incorretas chegando ao atendente sem filtro | Não há ponto de validação humana para temas sensíveis | Human-in-the-loop em pontos de risco (camada 4 e seção 5) |

O harness é o conjunto de verificações e limites entre a pergunta do atendente e a resposta confiável. Ele não é um componente único — é uma sequência de camadas, cada uma com responsabilidade e tipo de garantia próprios.

---

## 2. As cinco camadas do harness

| Camada | Tipo de garantia | Onde vive | O que faz |
|---|---|---|---|
| 1 · Validação de input | Determinística | `validator.ts` | Rejeita request malformado antes de qualquer custo de busca ou LLM |
| 2 · Structured output | Forçada pelo modelo | `completion.ts` | Força o modelo a responder no schema JSON, não em texto livre |
| 3 · Validação determinística | Determinística | `response-validator.ts` | Verifica os guardrails de enforcement via código; rejeita o que viola |
| 4 · Roteamento por confiança e risco | Determinística | `handler.ts` | Decide entre resposta direta, escalada automática ou HITL obrigatório |
| 5 · QueryLog | Auditoria | `handler.ts` → Contexto 4 | Registra toda decisão para revisão crítica posterior |

O princípio central: **quanto mais cedo o erro é barrado, mais barato é**. Validação de input (camada 1) custa quase nada; uma resposta incorreta que chega ao atendente custa confiança e pode causar dano operacional.

---

## 3. Camada 2 — Structured Outputs

### O problema que resolve

Hoje as respostas são texto livre. Quando o modelo "esquece" de incluir a fonte, nada impede a resposta de seguir. Structured outputs eliminam essa classe de erro forçando o modelo a responder em um formato JSON que **deve** ser preenchido — campos ausentes são rejeitados antes de a resposta sair.

### O schema obrigatório

O modelo é configurado (via `response_format` do Azure OpenAI com JSON schema) para sempre retornar:

```typescript
// src/shared/types.ts — schema enviado ao modelo como response_format
{
  answer: string,                    // conteúdo da resposta, sem marcação de UI
  source_document: {                 // null APENAS quando escalation_signal != null
    name: string,                    // ex: "PROC-042-v2"
    version: string,                 // ex: "2.0"
    section: string,                 // ex: "seção 2"
    date: string                     // ex: "10/11/2023"
  } | SourceCitation[] | null,
  confidence_score: number,          // 0–1, auto-reportado pelo modelo
  escalation_signal: {
    channel: EscalationChannel,      // enum fechado de 4 valores
    reason: string
  } | null,
  informal_source_warning: boolean,
  conflict_presentation: {
    version_a: { value, source, informal_source },
    version_b: { value, source, informal_source }
  } | null
}
```

### O que structured output garante e o que não garante

| Structured output **garante** | Structured output **não garante** |
|---|---|
| Os campos existem no JSON | Que o conteúdo dos campos esteja correto |
| O tipo de cada campo está certo | Que a fonte citada seja a fonte real do conteúdo |
| O enum de `channel` tem valor válido | Que o `confidence_score` reportado seja honesto |
| O JSON é parseável | Que o `answer` não contenha alucinação |

Essa distinção é o motivo de a camada 3 existir. Structured output resolve o problema de **campos faltantes** (a fragilidade identificada nos testes), mas não resolve o problema de **conteúdo incorreto**. Por isso a validação determinística vem depois.

---

## 4. Camada 3 — Validação determinística dos guardrails

A camada 3 aplica os guardrails de enforcement via código definidos em `prompts/guardrails.md`. O `response-validator.ts` recebe o structured output já parseado e verifica as invariantes — rejeitando respostas que violam, mesmo que estejam bem formadas.

| Invariante verificada | Guardrail | Ação se violado |
|---|---|---|
| `source_document` preenchido quando `escalation_signal = null` | G-D01 | Rejeita — HTTP 422 |
| `source_document` com os quatro campos completos | G-D01 | Rejeita — HTTP 422 |
| Nenhum chunk com `doc_status = histórico` foi usado como fonte | G-D02, G-N02 | Rejeita — re-processa sem o chunk |
| `conflict_presentation` preenchida quando conflito foi detectado | G-N04 | Rejeita — força apresentação de ambas versões |
| `escalation_signal = null` e `source_document = null` nunca simultâneos | invariante de estado | Rejeita — HTTP 500 |
| `informal_source_warning = true` quando algum chunk fonte é informal | G-N05, C-P05 | Corrige o campo antes de seguir |
| `channel` é um dos quatro valores do enum | G-D04, C-P07 | Rejeita — schema Zod |

A camada 3 é a diferença entre "o modelo tentou seguir as regras" e "as regras foram garantidas". Os guardrails de enforcement via prompt (G-D03, G-N01, G-N03, G-N07, G-F01, G-F04) não são verificáveis aqui — eles dependem do conteúdo gerado e são cobertos pelos VCs de regressão e pela revisão crítica (seção 6).

---

## 5. Camada 4 — Human-in-the-Loop (HITL)

### O princípio

Nem toda resposta precisa de validação humana — isso anularia o ganho de tempo do assistente. O HITL é reservado para os casos onde o **risco da decisão** justifica o custo da validação. O harness define esses pontos com base em dois eixos: confiança da resposta e sensibilidade do tema.

### Matriz de roteamento

| | Tema não sensível | Tema sensível |
|---|---|---|
| **Alta confiança** (`≥ 0.75`) | Resposta direta ao atendente | Resposta direta + flag de auditoria prioritária |
| **Baixa confiança** (`< 0.75`) | Escalada automática (sem HITL) | **HITL obrigatório** — supervisor valida antes de enviar |

### O que é um "tema sensível"

Temas onde uma resposta incorreta tem consequência regulatória, financeira ou de segurança. Definidos pela lista abaixo, mantida pelo Gestor do Corpus:

| Tema sensível | Por quê | Incidente relacionado |
|---|---|---|
| Carga perigosa (devolução, frete, prazo, ANTT) | Consequência regulatória e de segurança | INC-01 |
| Cálculo de frete com valor acima de R$ 10.000 | Consequência financeira direta | — |
| Devolução de carga (elegibilidade, prazo, exceções) | Risco de orientar fora da POL-001 | INC-01 |
| Penalidades e créditos de SLA | Consequência contratual com o cliente | — |
| Qualquer resposta baseada exclusivamente no FAQ informal | Fonte não validada por Compliance | INC-01 |

### Como o HITL funciona no fluxo

1. O `handler.ts` calcula confiança e verifica se o tema está na lista de sensíveis.
2. Se baixa confiança **e** tema sensível: a resposta não vai ao atendente. Entra na fila de validação do supervisor com a resposta proposta, os chunks usados e o `confidence_score`.
3. O supervisor tem três ações: **aprovar** (envia ao atendente como está), **editar** (corrige e envia), ou **rejeitar** (substitui por escalada).
4. A decisão do supervisor é registrada no `QueryLog` — permitindo medir a taxa de aprovação e identificar padrões de erro do assistente.

### Por que HITL não é "validação humana de tudo"

Com 320 chamados/dia e ~60% de consulta documental, validar 100% das respostas exigiria uma equipe dedicada — anulando o propósito do assistente. O HITL seletivo concentra o esforço humano onde ele tem mais valor: respostas que o sistema já sabe que são arriscadas. O atendente continua recebendo respostas diretas para a maioria dos casos.

---

## 6. Revisão crítica do que foi gerado por IA

O harness não governa apenas as respostas do assistente — governa também o que a IA produz durante o desenvolvimento. O incidente do módulo de feedback (gerado com Copilot, sem Zod, com log de dados sensíveis) mostra que código gerado por IA precisa do mesmo rigor de revisão que qualquer código.

### Revisão de código gerado por IA

Todo código gerado por agente (Copilot, Claude Code) passa por checklist antes do merge:

| Verificação | Origem da regra | Como verificar |
|---|---|---|
| Usa Zod nas bordas (sem validação manual de tipo) | AGENTS.md — Coding Standards | Code review + lint |
| Sem `console.log` — usa `pino` | AGENTS.md — C-S03 | Lint automático (`no-console`) |
| Não loga dados sensíveis do atendente | AGENTS.md — Product Rules | Code review obrigatório por humano |
| Sem importação cruzada entre contextos | Bounded contexts | Lint de arquitetura (dependency-cruiser) |
| Sem `any` implícito | `tsconfig.json` strict | Compilação |
| Cobertura de teste ≥ 80% por módulo | AGENTS.md — Testing Standards | `vitest --coverage` no CI |

**Regra de ouro:** código gerado por IA não é confiável por padrão. O fato de compilar e passar nos testes existentes não significa que respeita as regras do projeto. O incidente do feedback compilava e funcionava — mas violava duas regras do AGENTS.md que nenhum teste cobria.

### Revisão crítica de respostas do assistente

Além da validação automática (camadas 1–3), o QА executa revisão crítica periódica:

- **Eval semanal** com os golden queries (`prompts/eval/golden-queries.json`) nos primeiros 30 dias, depois mensal
- **Amostragem do QueryLog**: revisão manual de uma amostra de respostas de alta confiança para detectar alucinação que passou pela validação determinística (a métrica de "0% de alucinação" só é verificável assim)
- **Análise de todos os casos HITL rejeitados**: cada rejeição do supervisor é um caso de erro do assistente que deve virar um VC de regressão ou um novo guardrail

### Revisão crítica de testes gerados por IA

Testes gerados por IA têm um risco específico: podem testar o comportamento errado com confiança. Um teste gerado que afirma `expect(response.source_document).toBeDefined()` passa — mas não verifica se a fonte é a correta. A revisão de testes verifica:

- O teste cobre o comportamento esperado pela spec, não apenas o comportamento atual do código
- Os casos de borda dos VCs de `requirements.md` estão cobertos
- Os testes de guardrail (G-N01, G-N03, etc.) usam os casos exatos dos incidentes como fixtures

---

## 7. Como o harness se conecta aos artefatos anteriores

| Artefato | Cenário | Papel no harness |
|---|---|---|
| ADR-0001 (GPT-4o) | 1 | O modelo que recebe o schema de structured output |
| ADR-0002 (context budget) | 1 | Limita o que entra no prompt antes da geração |
| ADR-0003 (documentos contraditórios) | 1 | Base da detecção de conflito na camada 3 |
| `prompts/guardrails.md` | 2 | Define as invariantes verificadas na camada 3 |
| `specs/query-endpoint/requirements.md` | 2 | Os VCs viram os testes de regressão da revisão crítica |
| AGENTS.md — Product Rules | 2 | A fonte das regras de revisão de código gerado por IA |
| `src/shared/types.ts` | 2 | O schema de structured output e validação |

---

## 8. Checklist de prontidão para a demonstração à diretoria

A demonstração é em 2 semanas. O harness deve estar funcional nos seguintes pontos antes dela:

- [ ] Structured output configurado no `completion.ts` — modelo nunca retorna texto livre
- [ ] `response-validator.ts` aplica as 7 invariantes da camada 3
- [ ] Matriz de roteamento (camada 4) implementada no `handler.ts` com a lista de temas sensíveis
- [ ] Fila de HITL funcional para os 5 atendentes-piloto (mesmo que manual no piloto)
- [ ] Checklist de revisão de código gerado por IA aplicado retroativamente ao módulo de feedback (corrigir o incidente do Zod + log)
- [ ] Eval com golden queries rodando — métrica de respostas incorretas medida e abaixo da linha de corte definida com a diretoria
- [ ] QueryLog registrando todas as decisões para demonstrar rastreabilidade

> **Recomendação ao Delivery Manager:** a demonstração deve mostrar não só o caminho feliz, mas o harness em ação — uma pergunta sobre carga perigosa sendo escalada, um conflito de versão sendo apresentado, e um caso HITL sendo validado pelo supervisor. A confiabilidade do sistema é o que está sendo demonstrado, não apenas a capacidade de responder.

---

*Documento gerado na fase de governança — NovaTech × DB1*
*Leitura relacionada: `prompts/guardrails.md`, `specs/query-endpoint/requirements.md`, AGENTS.md*
