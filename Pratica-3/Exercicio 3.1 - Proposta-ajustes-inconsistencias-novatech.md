# Proposta de Ajustes — Inconsistências Identificadas em Staging

> **Origem:** comparação entre avaliação manual e avaliação por IA das 6 respostas em staging
> **Fonte de verdade:** Anexo A — Documentação Simulada NovaTech
> **Objetivo:** propor ajustes concretos em três frentes — respostas do assistente, processo de avaliação humana e harness de governança
> **Prazo de referência:** demonstração à diretoria em 2 semanas

---

## 1. Resumo do diagnóstico

A comparação das avaliações revelou inconsistências em duas dimensões:

**Dimensão 1 — Respostas do assistente com defeito** (capturadas pela avaliação contra o Anexo A):
- 2 respostas parcialmente corretas (1 e 3) — erro de rastreabilidade ou roteamento
- 2 respostas incorretas (4 e 6) — alucinação e confiança indevida sobre temas sensíveis

**Dimensão 2 — Inconsistência no processo de avaliação** (capturada pela divergência entre avaliações):
- A avaliação manual classificou como "correta" 2 respostas que continham alucinações objetivas (4 e 6)
- A divergência sistemática (manual sempre mais permissiva) indica um viés de avaliação a corrigir

Esta proposta endereça as duas dimensões.

---

## 2. Ajustes nas respostas do assistente

Para cada resposta defeituosa, o ajuste especifica: o problema, a resposta corrigida e o mecanismo que deve garantir a correção.

### Ajuste R1 — Resposta sobre prazo de devolução standard

| Item | Detalhe |
|---|---|
| Problema | Cita POL-001 seção 3.2 (Exceções) para sustentar o prazo de 7 dias, que está na seção 3.1 (Prazo geral) |
| Resposta corrigida | Mesmo conteúdo, citando POL-001 v3.1, **seção 3.1** para o prazo e **seção 3.3** para o procedimento de abertura de chamado |
| Mecanismo de garantia | Validação de coerência entre afirmação e seção citada (ver Ajuste H3) |
| Severidade | Média — conteúdo certo, rastreabilidade quebrada |

### Ajuste R3 — Resposta sobre devolução de carga perigosa

| Item | Detalhe |
|---|---|
| Problema | Escala para "supervisor" quando a POL-001 seção 3.2 especifica o ramal 4500 — Gestão de Riscos |
| Resposta corrigida | Mesmo conteúdo, escalando para o **ramal 4500 — Gestão de Riscos** |
| Mecanismo de garantia | Tabela de canais de escalada por tema aplicada no enforcement (G-D04/C-P07) |
| Severidade | Média — conteúdo certo, canal genérico em vez de específico |

### Ajuste R4 — Resposta sobre carga danificada em trânsito

| Item | Detalhe |
|---|---|
| Problema | Afirma política inexistente com confiança Alta e sem fonte; inventa detalhes que não constam nem no FAQ informal |
| Resposta corrigida | "Não há documento normativo formal sobre tratamento de carga danificada em trânsito. Esse é um processo de sinistro, distinto de devolução. Encaminhe para sinistros@novatech.com.br — o tratamento passa pelo Jurídico." Confiança: Baixa. Fonte: declaração de ausência. |
| Mecanismo de garantia | Invariante determinística: confiança Alta exige fonte preenchida (ver Ajuste H1) + guardrail G-N07 (distinguir devolução de sinistro) |
| Severidade | Alta — alucinação sobre tema sensível |

### Ajuste R6 — Resposta sobre carga perigosa com frete expresso

| Item | Detalhe |
|---|---|
| Problema | Afirma "Sim" categórico com confiança Alta baseado apenas no FAQ informal, sobre tema regulatório sensível |
| Resposta corrigida | "Não há documento normativo formal que defina o envio de carga perigosa por frete expresso. O tema requer validação do Compliance. Encaminhe para o ramal 4500 — Gestão de Riscos." Confiança: Baixa. |
| Mecanismo de garantia | Guardrail G-N05 (nunca responder carga perigosa só com FAQ) forçando escalada no handler (ver Ajuste H2) |
| Severidade | Alta — afirmação de maior risco regulatório do conjunto |

---

## 3. Ajustes no harness de governança

Os defeitos das respostas indicam que o harness, embora desenhado, ainda não está aplicando as invariantes. Estes ajustes tornam o enforcement efetivo.

### Ajuste H1 — Invariante: confiança Alta exige fonte preenchida

**O que implementar:** no `response-validator.ts`, adicionar invariante que rejeita qualquer resposta com `confidence_score` alto e `source_document = null`. A combinação é contraditória por definição — não se pode ter alta confiança em uma afirmação sem fonte.

**Captura:** teria barrado a resposta 4 antes de chegar ao atendente.

**Tipo:** enforcement determinístico — camada 3 do harness.

```typescript
// response-validator.ts
if (response.confidence_score >= CONFIDENCE_THRESHOLD
    && !response.source_document
    && !response.escalation_signal) {
  throw new ValidationError(
    "H1: confiança alta sem fonte e sem escalada — estado inválido"
  );
}
```

### Ajuste H2 — Escalada forçada para carga perigosa baseada em fonte informal

**O que implementar:** no `handler.ts`, quando a pergunta é classificada como tema de carga perigosa e todos os chunks recuperados têm `documentClassification = informal`, forçar escalada para o ramal 4500 antes de chamar o LLM.

**Captura:** teria barrado a resposta 6.

**Tipo:** semi-determinístico — detecção semântica do tema + verificação estrutural da classificação da fonte (guardrail G-N05).

### Ajuste H3 — Validação de coerência entre afirmação e seção citada

**O que implementar:** quando viável, verificar que a seção citada no `source_document` realmente contém o conteúdo afirmado. Para casos onde o chunk recuperado tem metadado de seção, comparar a seção do chunk usado com a seção citada na resposta.

**Captura:** teria sinalizado a resposta 1 (citou 3.2, conteúdo veio da 3.1).

**Tipo:** determinístico parcial — aplicável onde os metadados de seção do chunk permitem a verificação.

**Limitação reconhecida:** nem toda afirmação é verificável estruturalmente. Onde não for, o caso fica para a revisão crítica (seção 4).

### Ajuste H4 — Tabela de canais de escalada por tema

**O que implementar:** centralizar a tabela de canais de escalada por tema no `handler.ts`, de modo que carga perigosa sempre resolva para o ramal 4500, sinistro para sinistros@novatech.com.br, e assim por diante — em vez de o LLM escolher o canal livremente.

**Captura:** teria corrigido a resposta 3 (escalou para supervisor em vez do ramal 4500).

**Tipo:** determinístico — mapa de tema para canal.

---

## 4. Ajustes no processo de avaliação humana

A inconsistência mais sutil não está nas respostas — está no fato de a avaliação manual ter validado duas alucinações. Estes ajustes corrigem o processo de revisão.

### Ajuste A1 — Checklist de verificação obrigatória por resposta

**Problema:** a avaliação manual confiou na plausibilidade das respostas 4 e 6. Respostas fluentes receberam menos escrutínio.

**Ajuste:** todo avaliador humano deve responder a três perguntas objetivas antes de classificar uma resposta como correta:

1. A afirmação central existe literalmente em um documento normativo (POL/PROC/SLA)? Se a fonte é o FAQ informal ou "nenhuma", a resposta não pode ser "correta" sobre tema sensível.
2. A seção citada realmente contém a afirmação? (abrir o documento e confirmar)
3. A confiança declarada é coerente com a fonte? (confiança Alta + fonte informal/ausente = incoerência)

Uma resposta só é "correta" se as três passam. Caso contrário, é no mínimo "parcialmente correta".

### Ajuste A2 — Tratamento especial para temas sensíveis

**Problema:** as duas respostas que a avaliação manual errou (4 e 6) eram sobre temas sensíveis (carga danificada e carga perigosa).

**Ajuste:** respostas sobre temas sensíveis (carga perigosa, devolução, penalidades de SLA, valores de frete acima de R$ 10.000) recebem revisão reforçada — verificação obrigatória contra o documento normativo, sem exceção. A plausibilidade da resposta não substitui a verificação.

### Ajuste A3 — Dupla avaliação para o conjunto de validação

**Problema:** a divergência entre as duas avaliações foi o que trouxe as alucinações à tona. Se houvesse só a avaliação manual, os erros teriam passado.

**Ajuste:** para o conjunto de golden queries que valida o assistente, manter duas avaliações independentes (humana + IA). Toda divergência entre elas é investigada com uma terceira verificação contra o Anexo A. A concordância não garante acerto, mas a divergência garante atenção ao ponto certo.

---

## 5. Plano de aplicação priorizado

Considerando a demonstração à diretoria em 2 semanas, os ajustes são priorizados por impacto e esforço.

| Prioridade | Ajuste | Frente | Esforço | Bloqueia demo? |
|---|---|---|---|---|
| 1 | H1 — Invariante confiança vs fonte | Harness | Baixo | Sim |
| 2 | H2 — Escalada forçada carga perigosa | Harness | Médio | Sim |
| 3 | H4 — Tabela de canais de escalada | Harness | Baixo | Não |
| 4 | A1 — Checklist de avaliação | Processo | Baixo | Sim |
| 5 | A2 — Revisão reforçada temas sensíveis | Processo | Baixo | Sim |
| 6 | R4, R6 — Corrigir respostas críticas | Resposta | Baixo | Sim |
| 7 | A3 — Dupla avaliação | Processo | Médio | Não |
| 8 | H3 — Coerência afirmação/seção | Harness | Alto | Não |
| 9 | R1, R3 — Corrigir respostas de rastreabilidade | Resposta | Baixo | Não |

**Mínimo para a demonstração:** os ajustes H1, H2, A1, A2 e a correção de R4/R6 garantem que as duas alucinações de maior risco (carga danificada e carga perigosa) não se repetem. Sem eles, a demonstração corre o risco de reproduzir ao vivo o erro mais grave — uma afirmação categórica sobre carga perigosa sem respaldo normativo.

---

## 6. Princípio que conecta os três ajustes

As inconsistências têm uma raiz comum: **informação não validada tratada como fato**. A resposta 4 inventou uma política; a resposta 6 promoveu o FAQ informal a fonte normativa; e a avaliação manual confiou na fluência das duas.

Os três tipos de ajuste atacam a mesma raiz por camadas diferentes:
- O ajuste de **resposta** corrige o output específico.
- O ajuste de **harness** impede que o output defeituoso chegue ao atendente, deterministicamente.
- O ajuste de **processo** garante que, se algo escapar do harness, a revisão humana tenha o método para detectar.

Nenhuma das três camadas é suficiente sozinha. O harness não captura tudo (H3 tem limitação reconhecida). A avaliação humana erra sob plausibilidade (foi o que aconteceu). Mas as três juntas, com a dupla avaliação fechando o ciclo, formam uma defesa em profundidade contra o modo de falha que define este conjunto de inconsistências.

---

*Proposta elaborada contra o Anexo A — NovaTech × DB1*
*Base: comparação das avaliações manual e por IA das 6 respostas em staging*
