# Linguagem Ubíqua — NovaTech Assistant

> **Localização no repo:** `docs/onboarding.md` — seção 2  
> **Público:** todo o time + agentes de IA (referenciado pelo AGENTS.md)  
> **Fonte dos termos:** Anexo A (documentação NovaTech) + discovery Cenário 1

---

## Por que este documento existe

Linguagem ubíqua é o vocabulário compartilhado que todo membro do time — desenvolvedor, QA, Product Specialist, Delivery Manager — e todo agente de IA usa da mesma forma, sem ambiguidade. Quando o código, as specs, os testes e os prompts usam os mesmos termos com os mesmos significados, os agentes geram outputs consistentes e os erros de interpretação diminuem.

**Regra de uso:** os termos definidos aqui são os termos obrigatórios. Sinônimos informais são proibidos no código, nas specs e nos prompts do projeto — mesmo que sejam mais coloquiais.

---

## Termos do domínio de logística

### Documentos e corpus

| Termo obrigatório | Proibido usar | Definição |
|---|---|---|
| **documento normativo** | "doc oficial", "arquivo", "política" (genérico) | Documento com número de código (POL-XXX, PROC-XXX, SLA-XXXX) e responsável formal. É a fonte de verdade para o assistente. |
| **documento informal** | "FAQ", "anotações do time" | Documento sem responsável formal, sem controle de versão e sem validação por Compliance — especificamente o FAQ-Atendimento neste projeto. |
| **chunk** | "trecho", "pedaço", "fragmento" | Unidade mínima de texto extraída de um documento e indexada com embedding. É o que o Azure AI Search retorna ao buscar. |
| **corpus** | "base de dados", "documentação", "base de conhecimento" | Conjunto completo de chunks indexados no Azure AI Search que o assistente pode usar para responder. |
| **fonte autoritativa** | "documento certo", "versão válida" | O documento definido na tabela de fontes autoritativas como a fonte de verdade para um tema específico. |
| **vigência** | "validade", "data de corte" | Estado formal de um documento: se está ativo para uso nas respostas do assistente. |
| **cobertura bloqueada** | "assunto proibido", "tema sem resposta" | Tema para o qual o assistente não deve tentar responder porque o documento regulador está ausente ou em revisão (ex: PROC-043). |

### Tipos de carga

| Termo obrigatório | Proibido usar | Definição |
|---|---|---|
| **carga especial** | "frete especial" (para a carga) | Carga com peso acima de 500 kg, sujeita ao cálculo do PROC-042. |
| **carga perigosa** | "produto perigoso", "carga ANTT" | Carga classificada nas classes 1 a 6 da ANTT (conforme Resolução ANTT nº 5.947/2021): explosivos, gases, líquidos inflamáveis, sólidos inflamáveis, oxidantes/peróxidos, substâncias tóxicas/infectantes. |
| **carga refrigerada** | "produto resfriado", "perecível" | Carga que requer controle contínuo de temperatura, monitorada por sensor IoT. Devolução tem regra específica. |
| **CT-e** | "nota fiscal de frete", "documento de transporte" | Conhecimento de Transporte Eletrônico — documento fiscal obrigatório em todo chamado de devolução. |

### Cálculo de frete

| Termo obrigatório | Proibido usar | Definição |
|---|---|---|
| **frete especial** | "frete pesado", "frete diferenciado" | Modalidade de frete para cargas acima de 500 kg, calculada pela fórmula do PROC-042: Valor base × Multiplicador regional × Fator de peso. |
| **valor base** | "preço base", "tarifa" | Tarifa publicada na tabela mensal de fretes (`frete-base-AAAAMM.xlsx`). Atualizada mensalmente pela Diretoria Comercial. |
| **multiplicador regional** | "fator regional", "coeficiente" | Fator aplicado sobre o valor base conforme a região de destino da carga (Sul, Sudeste, Centro-Oeste, Nordeste, Norte). Valores diferentes entre PROC-042 v1 e v2. |
| **fator de peso** | "coeficiente de peso", "fator kg" | Multiplicador aplicado conforme a faixa de peso da carga. Valores diferentes entre PROC-042 v1 e v2. |
| **frete reverso** | "frete de volta", "frete de devolução" | Frete cobrado do cliente quando devolve por desistência (carga correta, sem defeito). Calculado com os mesmos multiplicadores do frete original. |
| **desconto de volume** | "desconto de recorrência", "desconto por quantidade" | Desconto automático aplicado a partir de 8 fretes especiais/mês (5%) ou 15 fretes/mês (10%), conforme PROC-042 v2. |

### Clientes e SLA

| Termo obrigatório | Proibido usar | Definição |
|---|---|---|
| **tier** | "categoria", "tipo de cliente", "nível" | Classificação do cliente: Gold, Silver ou Standard. Determina os SLAs aplicáveis. Não existe tier Platinum. |
| **SLA de resposta** | "prazo de atendimento", "tempo de resposta" | Tempo máximo para o atendente dar o primeiro retorno ao cliente após abertura do chamado (mesmo que seja "estamos verificando"). |
| **SLA de resolução** | "prazo de resolução", "tempo de fechamento" | Tempo máximo para o problema do cliente ser efetivamente resolvido e o chamado encerrado. |
| **incidente crítico** | "chamado urgente", "prioridade máxima" | Chamado que atende a pelo menos um dos critérios da SLA-2024 seção 3: carga acima de R$ 100k com status desconhecido há mais de 6h, carga perigosa com irregularidade, 5+ chamados do mesmo cliente em 24h, risco à segurança de pessoas. |
| **violação de SLA** | "SLA estourado", "prazo perdido" | Situação em que o tempo de resposta ou resolução ultrapassou o limite contratual do tier do cliente. Sujeita a penalidades. |
| **crédito de SLA** | "ressarcimento", "desconto por atraso" | Compensação financeira aplicada ao cliente após violação de SLA: 5% na segunda violação, 10% na terceira ou mais no mesmo mês. |

### Devoluções e sinistros

| Termo obrigatório | Proibido usar | Definição |
|---|---|---|
| **devolução** | "retorno", "troca" | Solicitação do cliente para devolver mercadoria após entrega confirmada. Regida pela POL-001 v3.1. Prazo: 7 dias úteis. Canal: Portal do Cliente. |
| **sinistro** | "carga danificada", "avaria" (como processo) | Processo de tratamento de carga danificada durante o transporte — antes da entrega ou imediatamente após. Distinto de devolução. Canal: sinistros@novatech.com.br. Prazo de registro: 48h após recebimento. |
| **coleta reversa** | "busca", "retirada" | Operação de coleta da mercadoria no endereço do cliente para retorno ao centro de distribuição da NovaTech. Agendada em até 2 dias úteis após aprovação da devolução. |
| **devolução parcial** | "devolução de parte", "retorno parcial" | Devolução de volumes individuais quando a entrega envolveu múltiplos volumes. Cada volume segue o mesmo procedimento da devolução integral. |

### Atendimento e escalada

| Termo obrigatório | Proibido usar | Definição |
|---|---|---|
| **chamado** | "ticket", "solicitação", "ocorrência" (genérico) | Registro formal de uma demanda do cliente no sistema de atendimento. |
| **escalada** | "encaminhamento", "repasse" | Ação de transferir uma pergunta ou chamado para o canal ou pessoa competente quando o assistente não tem resposta segura ou o atendente não tem autonomia para resolver. |
| **canal de escalada** | "canal certo", "onde encaminhar" | Destino específico da escalada: ramal 4500 (Gestão de Riscos), Comercial, sinistros@novatech.com.br, supervisor de atendimento. |
| **reporte de feedback** | "reporte de erro", "feedback do atendente" | Registro formal do atendente informando que uma resposta do assistente estava incorreta, desatualizada ou incompleta. |

---

## Termos do sistema de IA (internos ao projeto)

Estes termos são usados no código, nos testes e nos prompts. Agentes de IA devem usá-los ao gerar código ou documentação para o projeto.

| Termo no código | Tipo | Contexto | Definição |
|---|---|---|---|
| `Query` | interface TypeScript | Contexto 2 | Pergunta do atendente com metadados da sessão |
| `RetrievedChunk` | interface TypeScript | Contexto 2 | Chunk retornado pelo Azure AI Search com score e metadados |
| `PromptContext` | interface TypeScript | Contexto 2 | Conjunto de chunks dentro do context budget |
| `AssistantResponse` | interface TypeScript | Contexto 2 | Resposta gerada com texto, citação e nível de confiança |
| `EscalationSignal` | interface TypeScript | Contexto 2 | Indicação de escalada com canal e motivo |
| `SourceCitation` | interface TypeScript | Contexto 2 | Citação estruturada: nome, versão, seção, data |
| `ConflictPresentation` | interface TypeScript | Contexto 2 | Duas versões conflitantes com fonte para cada uma |
| `DocumentStatus` | enum TypeScript | Contexto 1 | `vigente` \| `histórico` \| `pendente_classificação` \| `vigente_transitório` |
| `BlockedTopic` | interface TypeScript | Contexto 1 | Tema com cobertura bloqueada e canal de escalada |
| `FeedbackReport` | interface TypeScript | Contexto 4 | Reporte do atendente vinculado ao QueryLog |
| `QueryLog` | interface TypeScript | Contexto 4 | Registro da consulta com chunks e scores para auditoria |
| `confidenceScore` | `number` (0–1) | Contexto 2 | Score de similaridade do chunk mais relevante. Abaixo de 0.75: declarar ausência. |
| `contextBudget` | configuração | Contexto 2 | Distribuição de tokens: ~4K system prompt + ~8K chunks + pergunta + 3 turnos de histórico (ADR-0002) |
| `sourceDocument` | campo na resposta | Contexto 2 | Campo obrigatório em toda `AssistantResponse` — nunca omitido |

---

## Termos explicitamente proibidos

Estes termos não devem aparecer em código, specs, prompts ou documentação do projeto:

| Proibido | Use em vez disso | Motivo |
|---|---|---|
| "alucinação" | "resposta sem base no corpus" | Termo genérico de IA; no projeto o problema específico é responder sem fonte no corpus |
| "base de dados" | "corpus" ou "índice" | Ambíguo — pode confundir com banco de dados relacional |
| "IA" (genérico em código) | "assistente" ou o nome do serviço (GPT-4o, Azure OpenAI) | Impreciso para fins de rastreabilidade de erros |
| "temperatura" (do modelo) | `temperature` (somente em código, nunca em specs de produto) | Detalhe técnico que não pertence a specs de produto |
| "Platinum" | — | Tier inexistente na NovaTech desde 2022; uso causa confusão com o cliente |
| "document" (em português no código) | `Document` (em inglês, em código TypeScript) | O código é em inglês; misturar idiomas gera inconsistência |

---

*Leitura seguinte: `specs/query-endpoint/requirements.md` — spec do módulo principal*  
*Gerado na fase de estruturação — NovaTech × DB1*
