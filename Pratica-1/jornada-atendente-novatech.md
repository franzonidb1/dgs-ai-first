# Jornada do Atendente — Assistente de IA NovaTech

> Documento produzido na fase de discovery do projeto NovaTech × DB1  
> Base: mapeamento documental (Etapas 1–3) + dados operacionais simulados  
> Complementa: diagrama de fluxo gerado na Etapa 4

---

## Contexto operacional

| Indicador | Situação atual | Meta com o assistente |
|---|---|---|
| Fontes abertas por chamado | ~4 fontes | 1 (o assistente) |
| Tempo médio de busca | 12 minutos | < 2 minutos |
| Chamados que escalam ao supervisor | 15% | Reduzir com escalada orientada |
| Chamados com consulta documental | ~60% (192/dia) | Cobertos pelo assistente |

**Distribuição de temas (60% dos chamados):**
- Prazos de entrega — 35%
- Regras de frete — 25%
- Política de devolução — 20%
- Outros (tracking, seguro, compliance) — 20%

---

## 1. Fluxo principal

> Cenário: atendente recebe dúvida do cliente e o assistente tem resposta fundamentada no corpus.

### Passo 1 — Recebimento da dúvida

O atendente recebe o chamado do cliente via canal de atendimento (telefone, portal ou e-mail). Identifica que a dúvida envolve consulta à documentação interna — prazo, frete, devolução, SLA ou compliance.

**Antes do assistente:** abria em média 4 fontes diferentes (SharePoint, Confluence, pasta de rede, colega de trabalho) para encontrar a resposta.

### Passo 2 — Consulta ao assistente

O atendente abre o assistente integrado ao Microsoft Teams e digita a pergunta em linguagem natural, sem precisar saber em qual documento a informação está.

**Exemplos de perguntas:**
- "Qual o prazo de entrega para frete especial acima de 1 tonelada para o Nordeste?"
- "Cliente Gold está reclamando que não recebeu resposta em 3 horas — está dentro do SLA?"
- "Cliente quer devolver mercadoria entregue há 5 dias. É possível?"

### Passo 3 — Resposta com fonte citada

O assistente retorna a resposta em linguagem natural, acompanhada obrigatoriamente de:

- **Fonte:** nome do documento (ex: PROC-042-v2, SLA-2024, POL-001 v3.1)
- **Seção:** referência exata (ex: "seção 2, fator de peso")
- **Versão e data:** para que o atendente saiba se o documento é recente

**Exemplo de resposta esperada do assistente:**
> "Para frete especial acima de 1.001 kg e até 3.000 kg com destino ao Nordeste, o valor do frete é calculado com fator de peso 1,15 e multiplicador regional 1,5. O prazo de entrega é o prazo padrão da rota acrescido de 3 dias úteis para manuseio.
> *Fonte: PROC-042-v2, seção 2 — vigente para chamados a partir de 01/12/2023.*"

### Passo 4 — Validação pelo atendente

O atendente lê a resposta e avalia se faz sentido para o contexto do chamado. A citação de fonte permite que o atendente confira o trecho original se necessário.

**Se a resposta faz sentido:** avança para o Passo 5.  
**Se a resposta parece incorreta ou incompleta:** aciona o fluxo de fallback (seção 2) ou o fluxo de feedback (seção 3).

### Passo 5 — Uso no atendimento

O atendente utiliza a resposta para orientar o cliente. O chamado é encerrado com a resposta fundamentada na documentação oficial.

**Resultado esperado:** tempo de busca reduzido de 12 para menos de 2 minutos; resposta consistente e rastreável à fonte.

---

## 2. Fluxo de fallback

> Cenário: o assistente não encontra resposta no corpus, sinaliza baixa confiança, ou o atendente discorda da resposta recebida.

### Situação 2A — Assistente sinaliza ausência de resposta

O assistente não encontrou informação suficiente no corpus para responder com confiança. Nesse caso, **não inventa uma resposta** — sinaliza explicitamente o limite e orienta a escalada.

**Formato de sinalização esperado do assistente:**
> "Não encontrei informação suficiente no corpus para responder com segurança sobre [tema]. Recomendo escalar para [área responsável]."

**Destinos de escalada por tema — mapeados no discovery:**

| Tema | Canal de escalada | Motivo |
|---|---|---|
| Frete de carga perigosa acima de 500 kg | Ramal 4500 — Gestão de Riscos | PROC-043 em revisão pelo Compliance; sem regra publicada |
| Custo de frete reverso por desistência do cliente | Comercial | Referência ambígua na POL-001 v3.1 sobre qual versão do PROC-042 aplicar |
| Seguro de carga | Comercial | Sem documento normativo no corpus; percentuais variam por contrato |
| Carga danificada em trânsito | sinistros@novatech.com.br | Processo de sinistros não formalizado; tratamento pelo Jurídico |
| Exceções à política de devolução não listadas | Supervisor de atendimento | Casos fora do escopo da POL-001 v3.1 |

### Situação 2B — Atendente discorda da resposta

O assistente retornou uma resposta, mas o atendente percebe que ela não corresponde ao caso do cliente (ex: versão do contrato diferente, cliente com condição especial negociada, informação que o atendente sabe que mudou).

**Ação do atendente:**
1. Não usar a resposta do assistente sem verificação.
2. Consultar a fonte citada diretamente para confirmar.
3. Se a discordância persistir, escalar ao supervisor e acionar o fluxo de feedback (seção 3).

### Situação 2C — Tema com cobertura bloqueada

Alguns temas não devem ser respondidos pelo assistente enquanto a documentação não estiver estabilizada. O assistente reconhece esses temas e responde com orientação de escalada, sem tentar calcular ou inferir.

**Temas com cobertura bloqueada até regularização documental:**
- Frete de carga perigosa acima de 500 kg (aguarda PROC-043 revisado)
- Cálculo de frete reverso em contratos anteriores a 01/12/2023 (aguarda esclarecimento da POL-001)

---

## 3. Fluxo de feedback

> Cenário: o atendente identifica que a resposta do assistente estava errada, desatualizada, incompleta ou conflitante com o que o cliente informou.

### Passo 1 — Identificação do problema

O atendente percebe um dos seguintes problemas na resposta do assistente:
- **Erro factual:** valor, prazo ou regra incorretos em relação à documentação oficial
- **Informação desatualizada:** o documento fonte foi revisado mas o assistente ainda usa a versão anterior
- **Resposta incompleta:** o assistente respondeu parte da pergunta, omitindo um caso relevante
- **Conflito com realidade:** o cliente apresenta contrato ou condição que contradiz a resposta

### Passo 2 — Reporte estruturado

O atendente aciona o mecanismo de feedback diretamente no Teams, preenchendo:

- **Chamado de origem:** número do chamado que gerou a consulta
- **Pergunta feita:** texto exato digitado no assistente
- **Resposta recebida:** cópia da resposta do assistente (incluindo fonte citada)
- **Problema identificado:** descrição do erro ou inconsistência
- **Informação correta (se souber):** referência à fonte ou versão correta

### Passo 3 — Investigação

A equipe responsável pelo tema avalia o reporte:

| Tipo de problema | Área responsável |
|---|---|
| Erro em cálculo de frete | Diretoria Comercial |
| Informação de SLA incorreta | Diretoria de Operações |
| Política de devolução desatualizada | Diretoria de Operações |
| Conflito entre versões de documento | Área que detém o documento (Ops / Compliance / Comercial) |
| FAQ com informação errada | Atendimento + validação pelo dono do tema |

### Passo 4 — Correção na fonte

A equipe responsável corrige o documento de origem — não apenas a resposta do assistente. A correção pode envolver:
- Atualização do documento normativo (PROC, POL, SLA)
- Arquivamento da versão obsoleta
- Publicação de nota de esclarecimento para casos ambíguos

### Passo 5 — Re-indexação

A correção chega ao assistente no próximo ciclo de atualização do índice. O ciclo segue o processo mensal de revisão documental já existente na NovaTech (Operações, Compliance e Comercial).

**Loop fechado:** a resposta corrigida passa a estar disponível no assistente a partir da próxima re-indexação. O atendente que reportou o erro recebe confirmação de que a correção foi aplicada.

---

## 4. Guardrails de comportamento do assistente

Guardrails são restrições de comportamento que o assistente deve observar independentemente da pergunta recebida. Eles protegem o atendente e o cliente de respostas que parecem corretas mas podem causar dano operacional, financeiro ou regulatório.

---

### Guardrail 1 — Nunca informar prazo ou valor sem documento que o sustente

**Regra:** O assistente não deve estimar, inferir ou calcular prazos de entrega ou valores de frete sem que a informação esteja explicitamente presente no corpus indexado.

**Gatilho:** Pergunta sobre prazo ou valor quando a informação não está no corpus, ou quando o corpus tem documentos conflitantes sem resolução formal.

**Comportamento esperado:**
> ❌ "O prazo estimado é de aproximadamente 5 dias úteis."  
> ✅ "Não encontrei o prazo para essa rota no corpus atual. Consulte o supervisor ou o setor de Operações para confirmar."

**Motivação (mapeada no discovery):** O PROC-042 v1 e v2 coexistem com multiplicadores diferentes. Se o assistente tentar calcular sem saber qual versão aplicar, pode gerar um valor de frete incorreto com aparência de resposta confiável.

---

### Guardrail 2 — Nunca responder sobre carga perigosa com base apenas no FAQ

**Regra:** Perguntas que envolvam carga perigosa — devolução, frete, prazo, documentação ANTT — não devem ser respondidas com base no FAQ informal. O assistente deve citar exclusivamente documentos normativos (POL, PROC, SLA) ou acionar escalada.

**Gatilho:** Qualquer pergunta que mencione "carga perigosa", "ANTT", "classe de risco", "explosivo", "inflamável", "tóxico" ou categorias equivalentes.

**Comportamento esperado:**
> ❌ "Sim, pode enviar carga perigosa com frete expresso — só precisa de autorização do Compliance."  
> ✅ "Frete de carga perigosa acima de 500 kg segue regras específicas que estão em revisão pelo Compliance. Encaminhe ao ramal 4500 — Gestão de Riscos — para tratamento individual. *Fonte: POL-001 v3.1, seção 3.2.*"

**Motivação (mapeada no discovery):** O FAQ contém orientações sobre carga perigosa (itens 3 e 32) que não foram validadas por Compliance ou Operações. O PROC-043, que rege o frete de carga perigosa acima de 500 kg, está em revisão ativa e não deve ser citado até publicação da versão revisada.

---

### Guardrail 3 — Sempre distinguir devolução de sinistro antes de orientar o atendente

**Regra:** Quando o atendente perguntar sobre "carga danificada", o assistente deve identificar se se trata de devolução (decisão pós-entrega do cliente) ou sinistro (avaria em trânsito com responsabilidade da NovaTech) antes de fornecer qualquer orientação — os fluxos, prazos e canais são completamente diferentes.

**Gatilho:** Perguntas contendo "carga danificada", "mercadoria com avaria", "produto chegou quebrado", "embalagem violada".

**Comportamento esperado:**
> "Para orientar corretamente, preciso entender: a carga foi danificada durante o transporte (antes da entrega) ou o cliente quer devolver após ter recebido?  
> - **Danificada em trânsito:** registrar ocorrência em até 48h — encaminhar para sinistros@novatech.com.br  
> - **Devolução após entrega:** prazo de 7 dias úteis — abertura no Portal do Cliente. *Fonte: POL-001 v3.1, seção 3.3.*"

**Motivação (mapeada no discovery):** O FAQ (item 38) e a POL-001 v3.1 descrevem processos distintos com prazos, canais e áreas responsáveis completamente diferentes. Confundir os dois fluxos pode fazer o cliente perder o prazo correto ou ser direcionado ao canal errado.

---

### Guardrail 4 — Nunca confirmar desconto sem verificar o limiar correto

**Regra:** O assistente não deve confirmar ou negar desconto de frete com base no FAQ. Deve sempre usar o PROC-042 v2 como fonte, que define limiares diferentes dos que o time conhece informalmente.

**Gatilho:** Perguntas sobre desconto de frete especial, volume de fretes, benefício por recorrência.

**Comportamento esperado:**
> ❌ "Desconto automático só para quem tem mais de 10 fretes especiais por mês."  
> ✅ "O desconto automático é aplicado a partir de 8 fretes especiais por mês (5% sobre o multiplicador regional). Acima de 15 fretes/mês, o desconto é de 10%. Para outros casos, encaminhar ao Comercial com justificativa. *Fonte: PROC-042-v2, seção 4.*"

**Motivação (mapeada no discovery):** O FAQ (item 45) informa limiar de 10 fretes, enquanto o PROC-042 v2 define 8 fretes como limiar correto. Clientes com 8 ou 9 fretes/mês estão sendo indevidamente privados do desconto toda vez que o atendente segue o FAQ. Esse erro tem impacto financeiro recorrente e mensurável.

---

## 5. Resumo dos fluxos e guardrails

```
CHAMADO RECEBIDO
      │
      ▼
ATENDENTE CONSULTA O ASSISTENTE (Teams)
      │
      ├─── Resposta encontrada com fonte ──► Atendente valida ──► Usa no atendimento ──► Chamado encerrado
      │                                              │
      │                                     Resposta incorreta
      │                                              │
      │                                              ▼
      │                                     FLUXO DE FEEDBACK
      │                                     Flag no Teams → Investigação → Correção → Re-indexação
      │
      └─── Sem resposta / tema bloqueado ──► FLUXO DE FALLBACK
                                             Escalada orientada por tema
                                             (ramal 4500 / Comercial / sinistros@novatech / supervisor)

GUARDRAILS ATIVOS EM TODOS OS FLUXOS:
  G1 — Nunca informar prazo ou valor sem documento que o sustente
  G2 — Nunca responder sobre carga perigosa com base apenas no FAQ
  G3 — Sempre distinguir devolução de sinistro antes de orientar
  G4 — Nunca confirmar desconto sem verificar o limiar correto (PROC-042 v2)
```

---

*Documento produzido na fase de discovery do projeto NovaTech × DB1*  
*Complementa: diagrama-jornada-atendente (Etapa 4) e analise-inconsistencias-novatech.md (Etapa 2)*  
*Próximo passo: Exercício 1.3 — Especificação de requisitos de RAG (ponto de vista do produto)*
