# Exercício 2.1 — Recorte de Domínio e Spec de Produto (SDD)
**Papel:** Product Specialist  
**Projeto:** NovaTech Assistant  
**Fase:** Cenário 2 — Estruturação  

---

## Parte 1 — Mapa de Bounded Contexts

### Visão geral

O projeto NovaTech Assistant toca quatro subdomínios de negócio. O contexto de **Atendimento ao Cliente** é o núcleo do sistema — é onde o assistente IA opera. Os outros quatro fornecem o conhecimento de domínio que o assistente consulta, mas nunca executa diretamente.

```
┌─────────────────────────┐     ┌─────────────────────────┐
│   Gestão Documental     │     │    Contratos e SLA       │
│                         │     │                          │
│  POL, PROC, SLA, FAQ    │     │  Tiers Gold/Silver/Std   │
│  Versionamento de docs  │     │  Prazos de resposta      │
│  Vigência/contradições  │     │  Incidentes críticos     │
│                         │     │                          │
│  Fora: criação de docs  │     │  Fora: negociação de     │
│  Fora: aprovação POL    │     │  contratos / penalidades │
└────────────┬────────────┘     └─────────────┬────────────┘
             │ fornece chunks                  │ define SLAs
             ▼                                 ▼
        ┌─────────────────────────────────────────┐
        │         Atendimento ao Cliente           │
        │            (contexto núcleo)             │
        │                                          │
        │  Perguntas de atendentes                 │
        │  Respostas com fonte citada              │
        │  Escopo do assistente IA                 │
        │                                          │
        │  Fora: ações operacionais diretas        │
        └───────────────┬──────────────────────────┘
             ▲          ▲
             │          │
┌────────────┴──────┐  ┌┴──────────────────────────┐
│ Logística de      │  │  Devoluções e Sinistros    │
│ Frete             │  │                            │
│                   │  │  Prazo (7 dias úteis)      │
│ Frete especial    │  │  Elegibilidade por carga   │
│ (> 500 kg)        │  │  Coleta reversa            │
│ Multiplicadores   │  │                            │
│ regionais         │  │  Fora: aprovação de        │
│ Carga perigosa    │  │  reembolsos / jurídico     │
│                   │  │                            │
│ Fora: frete       │  │                            │
│ padrão / rotas    │  │                            │
└───────────────────┘  └────────────────────────────┘
```

### Detalhamento dos contextos

#### Atendimento ao Cliente (contexto núcleo)
- **Dentro:** responder perguntas de atendentes sobre SLAs, fretes e devoluções; citar fontes; sinalizar contradições; sinalizar ausência de informação.
- **Fora:** executar ações no sistema, acessar dados em tempo real de clientes, criar ou atualizar documentos.
- **Relação com os outros contextos:** consome conhecimento dos quatro contextos satélite via pipeline de RAG. Não gerencia nenhum deles.

#### Gestão Documental
- **Dentro:** indexação, versionamento e rastreabilidade de POL, PROC, SLA e FAQ.
- **Fora:** criação, aprovação ou publicação de políticas.
- **Relação:** fornece os chunks que o assistente recupera. A decisão sobre qual versão é vigente é feita aqui (metadado de vigência — ADR-0003).

#### Contratos e SLA
- **Dentro:** definição dos três tiers (Gold, Silver, Standard), prazos de resposta/resolução, critérios de incidente crítico.
- **Fora:** negociação de contratos, aplicação de penalidades contratuais.
- **Relação:** o assistente consulta este contexto para responder perguntas sobre prazos e classificação de clientes.

#### Logística de Frete
- **Dentro:** fórmula de frete especial (> 500 kg), multiplicadores regionais, condições para carga perigosa.
- **Fora:** cálculo de frete padrão (< 500 kg — sem documentação na base atual), roteirização de entregas.
- **Relação:** o assistente consulta para calcular ou explicar fretes especiais.

#### Devoluções e Sinistros
- **Dentro:** prazo geral (7 dias úteis), elegibilidade por tipo de carga, procedimento de coleta reversa, custos de devolução.
- **Fora:** aprovação de reembolsos, processo jurídico de sinistros, negociação fora do prazo.
- **Relação:** o assistente consulta para responder sobre política de devolução.

---

## Parte 2 — Linguagem Ubíqua

> Estes termos devem ser usados de forma idêntica por humanos e agentes (LLM) em todo o projeto. Ambiguidade aqui gera respostas erradas do assistente.

### Classificação de clientes

| Termo | Definição canônica | Armadilha |
|---|---|---|
| **Tier** | Classificação de cliente NovaTech. Existem exatamente **3**: Gold, Silver e Standard. | Nunca usar "Platinum" ou qualquer outro nome — não existe. |
| **Gold** | Cliente com contrato anual > R$ 500.000 OU > 200 operações/mês. | "Gold" é um tier de cliente, não o metal. |
| **Silver** | Cliente com contrato entre R$ 100.000–500.000 OU 50–200 operações/mês. | — |
| **Standard** | Todos os demais clientes. | — |

### Tipos de carga

| Termo | Definição canônica | Armadilha |
|---|---|---|
| **Carga perigosa** | Exclusivamente cargas classificadas nas **classes 1 a 6 da ANTT** (Resolução nº 5.947/2021): explosivos (1), gases (2), líquidos inflamáveis (3), sólidos inflamáveis (4), oxidantes (5), substâncias tóxicas (6). | Nunca usar para "carga difícil de manusear", volumosa ou frágil. |
| **Frete especial** | Modalidade de frete para cargas com **peso acima de 500 kg**. Usa fórmula: Valor base × Multiplicador regional × Fator de peso. | Não confundir com "frete expresso" (prazo) ou "frete diferenciado". |
| **Frete expresso** | Modalidade com **prazo reduzido** — conceito distinto de frete especial (que é sobre peso). Requer autorização do Compliance para carga perigosa. | — |

### Processo de atendimento

| Termo | Definição canônica | Armadilha |
|---|---|---|
| **CT-e** | Conhecimento de Transporte Eletrônico — identificador único da operação logística. | Nunca substituir por "número do pedido" ou "nota fiscal". |
| **Incidente crítico** | Chamado que atende ao menos 1 critério da SLA-2024 seção 3: carga > R$ 100k desaparecida > 6h; carga perigosa com irregularidade; > 5 chamados do mesmo cliente em 24h sobre o mesmo problema; risco à segurança de pessoas. | Não confundir com "chamado urgente" ou "prioridade alta" — esses não têm definição formal. |
| **Dias úteis** | Exclui sábados, domingos e **feriados nacionais**. | Contexto: prazos de devolução e SLA de chamados gerais. SLA de incidentes críticos para Gold **não pausa** fora do horário comercial. |
| **Coleta reversa** | Retirada da mercadoria devolvida no endereço do cliente pela NovaTech. | Não é sinônimo de "devolução" (o processo completo). |

### Hierarquia de fontes

| Termo | Definição canônica |
|---|---|
| **Documento normativo** | POL-xxx e PROC-xxx — fonte primária, de uso obrigatório. Em contradição entre versões, priorizar a mais recente com metadado de vigência. |
| **Documento contratual** | SLA-xxxx — compromisso formal com o cliente. Mesmo peso de documento normativo. |
| **FAQ** | Documento informal, não validado por Compliance. Fonte secundária; nunca usar como única fonte para informação crítica (carga perigosa, valores, penalidades). |

---

## Parte 3 — requirements.md (Query Endpoint)

```
specs/query-endpoint/requirements.md
Versão: 1.0
Autor: Product Specialist
Status: aguardando revisão do Tech Lead
```

### 1. Outcomes — o que o usuário consegue fazer

- **O-1:** O atendente consulta uma regra de negócio (SLA, frete, devolução) e recebe uma resposta em menos de 30 segundos (medida até o primeiro token visível), com a fonte identificada.
- **O-2:** Quando duas fontes contradizem (ex: PROC-042 v1 vs v2), o atendente vê ambas as versões e qual tem vigência atual, para tomar a decisão certa.
- **O-3:** Quando a pergunta não tem cobertura documental, o atendente recebe uma sinalização clara de "informação não encontrada" — nunca uma resposta inventada.
- **O-4:** O atendente pode fazer perguntas que cruzam domínios (ex: "prazo de devolução + carga perigosa") e receber uma resposta coerente que integre as fontes corretas de cada contexto.

### 2. Scope boundaries — o que está dentro e fora

**Dentro do escopo** (contexto Atendimento ao Cliente):
- Responder perguntas sobre SLAs, fretes especiais e política de devoluções com base nos documentos indexados.
- Citar a fonte de cada informação (documento, seção, versão).
- Sinalizar documentos contraditórios e indicar qual tem vigência.
- Sinalizar quando a informação não está disponível na base documental.
- Manter histórico de até 3 turnos (3 pares pergunta+resposta) para contexto conversacional.
- Indicar quando a fonte é informal (FAQ), com disclaimer explícito na resposta.

**Fora do escopo:**
- Executar ações no sistema (abrir chamados, registrar ocorrências, dar descontos).
- Acessar dados do cliente em tempo real (status de tracking, histórico de contratos).
- Atualizar documentos da base (escopo do pipeline de ingestão).
- Responder sobre frete padrão (< 500 kg) — documentação não disponível na base atual.
- Fornecer aconselhamento jurídico (sinistros, penalidades contratuais).

### 3. Constraints — restrições não negociáveis

- **C-1:** Latência: resposta ao atendente em até 30 segundos (p95), medida do envio da pergunta até o primeiro token visível no Teams.
- **C-2:** Groundedness: o assistente nunca afirma algo que não está em nenhum chunk recuperado. Quando a fonte é o FAQ (informal), a resposta deve incluir disclaimer. Alucinação = zero de tolerância.
- **C-3:** Citação obrigatória: toda resposta factual deve incluir ao menos uma referência de fonte (documento + seção).
- **C-4:** Atualização da base documental em até 24h após publicação de novo documento no SharePoint.
- **C-5:** Stack: Azure OpenAI (GPT-4o), Azure AI Search, Azure Functions, TypeScript — conforme ADR-0001.
- **C-6:** Context budget: system prompt ~4K tokens, chunks ~8K (5 chunks de ~1.5K), histórico de 3 pares pergunta+resposta — conforme ADR-0002.

### 4. Prior decisions — ADRs que afetam este módulo

- **ADR-0001:** Modelo LLM: Azure OpenAI GPT-4o. Justificativa: janela de 128K tokens e integração com ecossistema Microsoft da NovaTech.
- **ADR-0002:** Estratégia de contexto: budget fixo de ~12K tokens (4K system + 8K chunks + pergunta + histórico de 3 pares).
- **ADR-0003:** Documentos contraditórios: metadado de vigência no pipeline; prompt instrui priorização da versão mais recente; documentos obsoletos marcados, não excluídos.
- **ADR-0004:** Chunking: tabelas exigem estratégia específica de chunking (identificado no protótipo com ChromaDB).

### 5. Verification criteria — testável pelo QA

- **V-1:** Dado: "Qual o prazo de devolução?" → Esperado: resposta menciona "7 dias úteis" e cita POL-001 seção 3.1. Nunca mencionar número diferente.
- **V-2:** Dado: pergunta sobre "cliente Platinum" → Esperado: assistente informa que esse tier não existe, menciona Gold/Silver/Standard e cita SLA-2024.
- **V-3:** Dado: "Qual o multiplicador para o Sudeste?" → Esperado: assistente cita PROC-042-v2 (1.1) como vigente; pode mencionar contradição com v1 (1.0).
- **V-4:** Dado: pergunta sobre frete padrão (< 500 kg) → Esperado: assistente responde que a informação não está disponível na base. Nunca inventar valor.
- **V-5:** Dado: "Posso devolver carga perigosa?" → Esperado: não é elegível pelo processo padrão; orientar contato com Gestão de Riscos (ramal 4500). Nunca dizer "impossível em hipótese alguma".
- **V-5b:** Dado: chunk do FAQ-03 é recuperado junto com POL-001-B para a mesma pergunta → Esperado: assistente prioriza o documento normativo (POL-001-B) e menciona que o FAQ tem uma orientação prática diferente, com disclaimer de fonte informal.
- **V-6:** Latência: 95% das respostas em até 30s (primeiro token) em ambiente de staging com carga simulada de 10 req/min.
- **V-7:** Dado: resposta baseada no FAQ (fonte informal) → Esperado: resposta inclui disclaimer explícito indicando que a fonte é informal e não validada por Compliance.

---

## Parte 4 — Iteração com o Tech Lead

### Ambiguidades levantadas pelo Tech Lead

Após revisão do requirements.md v1.0, o Tech Lead identificou 4 ambiguidades:

**1. O-1: "menos de 30 segundos" — inclui streaming?**
> Se a resposta exibir progressivamente, o atendente começa a ler antes dos 30s, mas a resposta completa pode levar mais. Precisamos definir: o SLA é "primeiro token" ou "resposta completa"?

**2. C-2 conflita com FAQ como fonte**
> O FAQ-38 sobre carga danificada não tem respaldo em documento formal. Se o pipeline indexar o FAQ, o assistente pode responder com base nele e ainda cumprir C-2 tecnicamente. Decisão necessária: o FAQ entra na base? Com qual peso? Com disclaimer explícito?

**3. V-5 incompleto para o QA**
> O critério não define o que acontece quando o chunk do FAQ-03 (que menciona exceções autorizadas pelo setor de Riscos) é recuperado junto com o documento normativo. Qual é a resposta esperada quando as fontes contradizem na mesma pergunta?

**4. "3 turnos" — é 3 perguntas ou 3 pares?**
> 3 perguntas ≈ 300 tokens; 3 pares pergunta+resposta ≈ 1.200 tokens. A diferença é material para o context budget.

### Resoluções do Product Specialist

1. O SLA de 30s é sobre **primeiro token visível** — a experiência do atendente conta a partir do momento que ele começa a ler. Atualizado em C-1 e O-1.
2. O FAQ entra na base com metadado `source_type: informal`. O system prompt instrui o assistente a incluir disclaimer quando a fonte for informal. Adicionado como C-2 (atualizado) e V-7.
3. Quando FAQ e documento normativo contradizem, priorizar o normativo e mencionar a orientação do FAQ com disclaimer. Adicionado como V-5b.
4. "3 turnos" = **3 pares pergunta+resposta**. Atualizado em C-6.

---

*Próximo passo: Tech Lead escreve `specs/query-endpoint/plan.md` com base neste requirements.md v1.1.*
