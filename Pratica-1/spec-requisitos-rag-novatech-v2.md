# Especificação de Requisitos de Produto — Pipeline RAG
## Assistente de Atendimento NovaTech — Versão 2.0

> **Versão:** 2.0 — incorpora revisão crítica da v1.0  
> **Projeto:** NovaTech × DB1 — Assistente de IA para Atendimento  
> **Escopo:** Requisitos funcionais do pipeline RAG sob a perspectiva de produto  
> **Base:** Discovery documental (Etapas 1–3) + Jornada do atendente + Anexo A + Análise de gaps da v1.0  
> **Não cobre:** Arquitetura técnica, escolha de modelos, infraestrutura Azure  
> **Alterações em relação à v1.0:** Ver seção 0 — Registro de mudanças

---

## Seção 0 — Registro de mudanças em relação à v1.0

| ID gap v1 | Categoria | Resolução adotada na v2 | Requer validação NovaTech? |
|---|---|---|---|
| G-01 | Crítico | REQ-C01 reescrito com mecanismo de detecção de conflito definido (lista + heurística por metadado) | Não |
| G-02 | Crítico | REQ-C02 inclui dono operacional, SLA de atualização e processo de resolução de conflito entre áreas | Sim — definir papel de Gestor do Corpus |
| G-03 | Crítico | Novo REQ-C04 — Política de esclarecimento: quando o assistente pergunta vs. quando age | Sim — validar campos obrigatórios por tipo de pergunta |
| G-04 | Crítico | REQ-A01 reescrito com critério operacional de confiança definido | Não |
| A-01 | Ambiguidade | REQ-F01 define campo de metadado exato, quem edita e processo de curadoria inicial | Sim — provisionar campo no SharePoint |
| A-02 | Ambiguidade | REQ-U01 distingue indexação automática vs. validada por tipo de documento | Não |
| A-03 | Ambiguidade | REQ-R02 define mecanismo de acionamento do trecho original | Não |
| A-04 | Ambiguidade | REQ-F03 define critério de elegibilidade por item do FAQ | Sim — validar com Compliance |
| A-05 | Ambiguidade | REQ-R04 define papel de Gestor do Corpus e SLA de resposta ao reporte | Sim — nomear responsável |
| R-01 | Ausente | Novo REQ-S01 — Controle de acesso por perfil de usuário | Sim — definir perfis |
| R-02 | Ausente | Novo REQ-S02 — Escopo de idioma e variação regional | Não |
| R-03 | Ausente | Novo REQ-P01 — Requisitos de latência e volume | Sim — validar SLAs técnicos |
| R-04 | Ausente | Novo REQ-P02 — Comportamento em modo offline/degradado | Não |
| I-01 | Inconsistência | REQ-U02 e REQ-U04 reconciliados via metadado de "disposição transitória" | Não |
| I-02 | Inconsistência | PROC-042 v1 reconhecido como caso especial com escopo temporal — tensão com REQ-F01 resolvida | Não |

> **Decisões pendentes de validação pela NovaTech estão marcadas com** ⚠️ **ao longo do documento.**

---

## Como ler este documento

Esta especificação descreve **o que o sistema deve fazer**, não como deve ser implementado tecnicamente. Cada requisito é formulado de forma que um profissional de QA consiga escrever um caso de teste sem pedir esclarecimentos adicionais.

**Estrutura de cada requisito:**
- **O que o sistema deve fazer** — comportamento esperado
- **Por quê** — motivação rastreada ao discovery ou à jornada
- **Como testar** — critério de aceitação verificável
- **Exemplo concreto** — caso real da documentação NovaTech

**Papel definido nesta spec — Gestor do Corpus:**  
Papel operacional responsável por manter a tabela de fontes autoritativas, aprovar inclusões de documentos no corpus, e investigar reportes de erro do time de atendimento. ⚠️ A NovaTech deve nomear a pessoa ou equipe que ocupa esse papel antes do go-live. Sugestão: analista de Operações ou Compliance com dedicação parcial.

---

## Seção 1 — Fontes de dados: o que deve e o que não deve ser indexado

### REQ-F01 — Somente documentos com status de vigência confirmado devem ser indexados como fonte autoritativa

**O que o sistema deve fazer:**  
Antes de indexar qualquer documento, o pipeline deve verificar se o documento possui o campo de metadado `doc_status` preenchido com um dos valores válidos: `vigente`, `histórico`, `pendente_classificação`, ou `vigente_transitório` (ver REQ-U04). Documentos sem esse campo ou com valor `pendente_classificação` não devem ser indexados como fonte autoritativa — ficam acessíveis apenas para o Gestor do Corpus via interface de administração.

**Campo de metadado obrigatório:**

| Campo | Valores válidos | Onde vive | Quem pode editar |
|---|---|---|---|
| `doc_status` | `vigente` / `histórico` / `pendente_classificação` / `vigente_transitório` | Coluna customizada no SharePoint / propriedade de página no Confluence | Gestor do Corpus + responsável da área dona do documento |

⚠️ **Decisão pendente NovaTech:** Provisionar a coluna `doc_status` no SharePoint e a propriedade equivalente no Confluence antes do início da indexação. Estimar ~800 documentos existentes que precisarão ser classificados no processo de curadoria inicial (ver pré-condições de go-live).

**Curadoria inicial:** Documentos existentes no SharePoint e Confluence sem o campo `doc_status` devem ser classificados pelo Gestor do Corpus em parceria com as áreas responsáveis antes da primeira indexação. O processo de curadoria inicial é pré-requisito de go-live — não é possível adiar para após a implantação.

**Caso especial — PROC-042 v1:** O PROC-042 v1 não possui status de vigência formal, mas deve ser mantido no corpus com status `vigente_transitório` e escopo temporal `chamados_anteriores_2023-12-01`. Esse metadado de escopo temporal o qualifica como vigente para um subconjunto definido de chamados, resolvendo a tensão com este requisito. Ver REQ-U04 para detalhes.

**Por quê:**  
O PROC-042 v1 e v2 coexistem sem marcação formal. Indexados sem distinção, o assistente pode usar qualquer um — gerando erro de 4,2% no cálculo de frete para cargas de 1.001–3.000 kg.

**Como testar:**  
Dado que PROC-042 v1 e v2 estão no SharePoint sem campo `doc_status` preenchido, o pipeline não deve indexá-los. Ao perguntar sobre fator de peso para 2.000 kg, o assistente deve responder: "Documentação sobre esse tema aguarda classificação pelo Gestor do Corpus. Consulte o Comercial." NÃO deve retornar valor calculado.

---

### REQ-F02 — Documentos indexados devem carregar conjunto mínimo de metadados obrigatórios

**O que o sistema deve fazer:**  
Todo documento indexado deve carregar os seguintes metadados, extraídos do cabeçalho do documento ou do campo correspondente no SharePoint/Confluence:

| Metadado | Campo de origem | Obrigatório? |
|---|---|---|
| `doc_nome` | Nome do arquivo / título da página | Sim |
| `doc_codigo` | Código do documento (ex: POL-001, PROC-042) | Sim |
| `doc_versao` | Número de versão (ex: 3.1, 2.0) | Sim |
| `doc_data_atualizacao` | Data da última atualização | Sim |
| `doc_area_responsavel` | Área responsável pelo documento | Sim |
| `doc_classificacao` | `normativo` / `informativo` / `informal` | Sim |
| `doc_status` | Ver REQ-F01 | Sim |
| `doc_substitui` | Código + versão do documento que este substitui | Quando aplicável |
| `doc_escopo_temporal` | Data de início e/ou fim de vigência | Quando aplicável |
| `doc_tem_disposicao_transitoria` | `true` / `false` | Sim |

Documentos com campos obrigatórios ausentes devem ser rejeitados pelo pipeline e sinalizados ao Gestor do Corpus para correção antes de nova tentativa de indexação.

**Como testar:**  
Dado que POL-001 v3.1 está sendo indexada sem o campo `doc_area_responsavel`, o pipeline deve rejeitar o documento e registrar o erro no log de indexação. O documento NÃO deve aparecer como fonte disponível para busca.

---

### REQ-F03 — O FAQ informal deve ter critério explícito de elegibilidade por item antes de ser indexado

**O que o sistema deve fazer:**  
O FAQ-Atendimento não deve ser indexado como corpus único. Cada item do FAQ deve ser avaliado individualmente pelo Gestor do Corpus segundo os seguintes critérios de elegibilidade para indexação como "fonte informal":

**Elegível para indexação como fonte informal:** Item cobre tema sem contrapartida normativa no corpus E não contradiz nenhum documento normativo indexado E foi revisado pelo responsável da área competente nos últimos 12 meses.

**Não elegível (excluir do corpus):** Item contradiz documento normativo indexado OU item cobre tema com documento normativo correspondente OU item está desatualizado em relação a alterações documentadas no discovery.

⚠️ **Decisão pendente NovaTech:** Validar com Compliance a elegibilidade dos 47 itens do FAQ antes da indexação. Com base no discovery, os seguintes itens já têm classificação recomendada:

| Item FAQ | Recomendação | Motivo |
|---|---|---|
| Item 8 (frete especial — conflito v1/v2) | Excluir | Contradiz ambas as versões do PROC-042 (erro factual sobre multiplicadores) |
| Item 15 (tier Platinum) | Indexar | Sem contrapartida normativa; alinhado ao SLA-2024 |
| Item 22 (seguro de carga) | Indexar com aviso | Sem documento normativo; percentuais não validados |
| Item 27 (tracking 5 dias) | Indexar com aviso | Sem procedimento normativo; critério de R$ 50k não validado |
| Item 38 (carga danificada) | Indexar com aviso | Descreve processo de sinistros (distinto de devolução); sem norma formal |
| Item 41 (SLA resposta vs resolução) | Indexar | Alinhado ao SLA-2024; redirecionar para o normativo como fonte primária |
| Item 45 (desconto de frete) | Excluir | Contradiz PROC-042 v2 no limiar de desconto (10 vs 8 fretes/mês) |
| Item 3 (carga perigosa — devolução) | Indexar com aviso | Alinhado à POL-001 v3.1 no canal de escalada (ramal 4500) |
| Item 32 (expresso + carga perigosa) | Excluir | Sem norma formal; PROC-043 em revisão |

Toda resposta gerada a partir de item do FAQ indexado deve incluir aviso visual: "⚠️ Fonte não validada por Compliance. Confirme com documento normativo antes de usar."

**Como testar:**  
Dado que o item 45 está excluído do corpus por contradizer o PROC-042 v2, ao perguntar "a partir de quantos fretes o cliente tem desconto?", o assistente deve retornar a resposta baseada no PROC-042 v2 (8 fretes/mês), não no FAQ. Se o PROC-042 v2 não estiver disponível, deve escalar ao Comercial.

---

### REQ-F04 — Documentos referenciados mas ausentes do corpus devem gerar registro de cobertura bloqueada

**O que o sistema deve fazer:**  
Durante a indexação, o pipeline deve identificar referências a documentos externos (ex: "ver PROC-043", "consultar PROC-088") e registrar automaticamente esses documentos em uma lista de cobertura bloqueada. Perguntas cujo tema principal depende de um documento com cobertura bloqueada devem acionar escalada — independentemente de existirem chunks relacionados no corpus.

A lista de cobertura bloqueada deve ser acessível ao Gestor do Corpus, que pode remover um tema da lista ao indexar o documento correspondente.

**Lista inicial de cobertura bloqueada (a ser mantida pelo Gestor do Corpus):**

| Tema | Documento ausente | Canal de escalada | Motivo do bloqueio |
|---|---|---|---|
| Frete de carga perigosa acima de 500 kg | PROC-043 | Ramal 4500 — Gestão de Riscos | Em revisão pelo Compliance |
| Interceptação de carga em trânsito | PROC-088 | Supervisor de atendimento | Não incluído no corpus inicial |
| Custo de frete reverso em contratos pré-2023 | Esclarecimento POL-001 | Comercial | Ambiguidade na POL-001 v3.1, seção 3.5 |

**Como testar:**  
Dado que PROC-043 está na lista de cobertura bloqueada, ao perguntar "qual o custo de frete para 800 kg de carga perigosa para o Norte?", o assistente deve responder com escalada ao ramal 4500. NÃO deve calcular valor usando os multiplicadores do PROC-042 v2, mesmo que esse chunk seja recuperado pela busca.

---

### REQ-F05 — Planilhas de referência com atualização mensal devem ter data de validade e alerta automático

**O que o sistema deve fazer:**  
Documentos classificados com ciclo de atualização mensal (ex: tabela de fretes base `frete-base-AAAAMM.xlsx`) devem ser indexados com metadado `doc_validade_ate` correspondente ao último dia do mês de vigência. No primeiro dia útil do mês seguinte, se uma versão mais recente não tiver sido indexada, o sistema deve: (a) bloquear o uso dessa fonte para cálculo de valores, e (b) enviar alerta ao Gestor do Corpus.

Enquanto a fonte estiver bloqueada por desatualização, o assistente deve responder perguntas de cálculo com: "A tabela de fretes base do mês corrente ainda não foi indexada. Consulte o sistema de cotação para obter o valor atualizado."

**Como testar:**  
Dado que a tabela de frete base de janeiro não foi indexada até 1º de fevereiro, ao perguntar sobre valor de frete nesse dia, o assistente NÃO deve usar a tabela de janeiro para calcular. Deve exibir a mensagem de tabela desatualizada e orientar o sistema de cotação.

---

## Seção 2 — Comportamento com documentos contraditórios

### REQ-C01 — O sistema deve detectar conflito por dois mecanismos combinados, em ordem de precedência

**O que o sistema deve fazer:**  
A detecção de conflito entre documentos ocorre por dois mecanismos aplicados nesta ordem:

**Mecanismo 1 — Lista explícita de pares conflitantes (precedência maior):**  
O Gestor do Corpus mantém uma lista de pares de documentos conhecidos como conflitantes, com o tema de conflito. Quando os dois documentos do par são recuperados para a mesma pergunta, o sistema detecta conflito automaticamente — sem precisar de análise semântica.

Lista inicial de pares conflitantes:

| Par | Tema do conflito |
|---|---|
| PROC-042 v1 + PROC-042 v2 | Fator de peso e multiplicadores regionais (fora do escopo de chamados pré-2023) |

**Mecanismo 2 — Heurística por metadado `doc_codigo` (precedência menor):**  
Se dois documentos com o mesmo `doc_codigo` e versões diferentes forem recuperados para a mesma pergunta, e ambos tiverem `doc_status = vigente`, o sistema detecta conflito automaticamente.

Quando conflito é detectado por qualquer mecanismo, o sistema não deve gerar resposta com valor ou regra específica. Deve apresentar as versões conflitantes com fonte para cada uma e orientar escalada à área definida na tabela de fontes autoritativas (REQ-C02).

**Como testar:**  
Dado que PROC-042 v1 e v2 estão na lista de pares conflitantes e ambos são recuperados para "qual o prazo adicional para frete especial?", o assistente deve apresentar:
- "PROC-042 v1 (mar/2023): prazo padrão + 2 dias úteis"
- "PROC-042 v2 (nov/2023): prazo padrão + 3 dias úteis"
- "Confirme com o Comercial qual versão aplica ao chamado deste cliente."

NÃO deve escolher uma versão e apresentá-la como resposta única.

---

### REQ-C02 — A tabela de fontes autoritativas deve ter dono definido, SLA de atualização e processo de resolução de conflito entre áreas

**O que o sistema deve fazer:**  
O pipeline deve suportar a configuração de uma tabela de fontes autoritativas por tema. Essa tabela é o mecanismo que resolve conflitos detectados pelo REQ-C01 quando a situação já foi analisada e decidida.

**Governança da tabela:**

| Aspecto | Definição |
|---|---|
| Dono operacional | Gestor do Corpus ⚠️ (papel a ser nomeado pela NovaTech) |
| SLA de atualização | Até 5 dias úteis após publicação de novo documento normativo |
| Processo de adição de tema | Gestor do Corpus propõe → área responsável aprova → Gestor registra |
| Resolução de conflito entre áreas | Quando Comercial e Operações discordarem, o Gestor do Corpus registra o tema como "conflito entre áreas" e aciona Diretoria para decisão em até 10 dias úteis. Enquanto não resolvido, o tema é tratado como cobertura bloqueada (REQ-F04). |

**Tabela inicial de fontes autoritativas (a ser validada com Comercial, Operações e Compliance):**

| Tema | Fonte autoritativa | Área responsável pela manutenção | Escopo |
|---|---|---|---|
| Classificação de tiers e SLA | SLA-2024 v2024.1 | Comercial + Operações | Todos os chamados |
| Cálculo de frete especial | PROC-042 v2 | Comercial | Chamados a partir de 01/12/2023 |
| Cálculo de frete especial (histórico) | PROC-042 v1 | Comercial | Chamados anteriores a 01/12/2023 |
| Prazo e processo de devolução | POL-001 v3.1 | Operações | Todos os chamados |
| Incidentes críticos e penalidades de SLA | SLA-2024 v2024.1 | Comercial + Operações | Todos os chamados |
| Desconto por volume de frete | PROC-042 v2, seção 4 | Comercial | Chamados a partir de 01/12/2023 |
| Devolução de carga perigosa | POL-001 v3.1, seção 3.2 | Operações + Compliance | Todos os chamados |

**Como testar:**  
Dado que a tabela está configurada com PROC-042 v2 como fonte para chamados a partir de 01/12/2023, e o atendente pergunta sobre fator de peso para 2.000 kg sem informar a data do chamado, o sistema deve solicitar a data (ver REQ-C04). Com data informada após 01/12/2023, deve usar fator 1.15 do PROC-042 v2.

---

### REQ-C03 — Conflitos sem resolução na tabela de fontes devem acionar escalada obrigatória

**O que o sistema deve fazer:**  
Se o tema da pergunta tem conflito detectado (REQ-C01) e não há entrada correspondente na tabela de fontes autoritativas (REQ-C02), o assistente deve:
1. Informar que há documentos conflitantes sobre o tema
2. Listar os documentos em conflito com suas versões e datas
3. Indicar o canal de escalada da área responsável pelo tema
4. NÃO apresentar qualquer valor calculado ou regra específica

**Como testar:**  
Dado que não há entrada na tabela de fontes autoritativas para "custo do frete reverso em contratos pré-2023", ao perguntar sobre esse tema, a resposta deve identificar o conflito, listar POL-001 v3.1 e PROC-042 (sem definir qual versão), e orientar escalada ao Comercial. NÃO deve apresentar valor calculado.

---

### REQ-C04 — O assistente deve pedir esclarecimento em situações definidas — e nunca em situações não definidas

**O que o sistema deve fazer:**  
O assistente pode solicitar esclarecimento ao atendente apenas quando uma informação obrigatória para gerar a resposta está ausente e está na lista de campos obrigatórios por tipo de pergunta. Fora dessa lista, o assistente não deve pedir esclarecimento — deve agir com o que tem ou acionar escalada.

**Campos obrigatórios por tipo de pergunta:**

| Tipo de pergunta | Campo obrigatório | Comportamento se ausente |
|---|---|---|
| Cálculo de frete especial | Data de abertura do chamado (para definir versão do PROC-042) | Solicitar data uma vez. Se não respondido: usar PROC-042 v2 com aviso explícito. |
| Consulta de SLA | Tier do cliente (Gold / Silver / Standard) | Solicitar tier uma vez. Se não respondido: apresentar os três tiers. |
| Devolução | Natureza (pós-entrega ou em trânsito) | Solicitar esclarecimento uma vez. Se não respondido: apresentar os dois fluxos (devolução e sinistro) com seus respectivos canais. |
| Desconto de frete | Volume de fretes especiais/mês do cliente | Solicitar volume uma vez. Se não respondido: escalar ao Comercial. |

**Regras gerais:**
- Máximo de 1 pergunta de esclarecimento por consulta.
- O assistente não faz perguntas encadeadas — se precisar de duas informações, solicita a mais crítica e usa comportamento padrão para a segunda.
- Se o atendente não responder ao pedido de esclarecimento na mesma sessão, o assistente usa o comportamento padrão definido na tabela acima.

⚠️ **Decisão pendente NovaTech:** Validar se a lista de campos obrigatórios está completa para os tipos de pergunta mais frequentes (prazos de entrega 35%, regras de frete 25%, devolução 20%).

**Como testar:**  
Dado que o atendente pergunta "qual o fator de peso para uma carga de 2.000 kg?" sem informar a data do chamado, o assistente deve solicitar: "Para qual data é o chamado? Isso define qual versão do procedimento de frete aplica." Se o atendente não responder, o assistente deve usar o PROC-042 v2 com o aviso: "Usando PROC-042 v2 (vigente para chamados a partir de 01/12/2023). Confirme a data do chamado se for anterior a essa data."

---

## Seção 3 — Comportamento quando não há resposta na base

### REQ-A01 — Quando não há resposta no corpus, o assistente declara ausência com base em critério operacional de confiança definido

**O que o sistema deve fazer:**  
O assistente declara ausência de resposta (e não tenta responder) quando qualquer uma das seguintes condições for atendida:

**Critério 1 — Score de similaridade insuficiente:**  
O chunk com maior score de similaridade retornado pela busca vetorial tem score inferior a 0.75 (em escala 0–1). Esse threshold é o valor inicial — deve ser calibrado com base nos primeiros 30 dias de operação em produção.

**Critério 2 — Nenhum chunk recuperado:**  
O mecanismo de busca não retorna nenhum chunk com score acima de 0.50.

**Critério 3 — Tema com cobertura bloqueada:**  
O tema da pergunta está na lista de cobertura bloqueada (REQ-F04) — independentemente dos scores de similaridade dos chunks recuperados.

**Critério 4 — Conflito sem resolução:**  
Conflito detectado (REQ-C01) sem entrada na tabela de fontes autoritativas (REQ-C02).

Em nenhuma dessas condições o assistente deve usar conhecimento geral do modelo de linguagem como substituto da resposta do corpus. O conhecimento geral sobre logística, fretes e políticas comerciais não é a política da NovaTech.

**Como testar:**  
Dado que não há documento sobre frete padrão (abaixo de 500 kg) no corpus e a busca retorna score máximo de 0.42, ao perguntar "qual o prazo de entrega para 200 kg de São Paulo para o Norte?", o assistente deve responder com mensagem de ausência e canal de escalada. NÃO deve estimar prazo com base em conhecimento geral.

---

### REQ-A02 — A mensagem de ausência deve ser específica ao tema e indicar o canal de escalada correto

**O que o sistema deve fazer:**  
A mensagem de ausência deve conter os três elementos obrigatórios:
1. Identificação do tema da pergunta
2. Motivo da ausência (documento não indexado / tema bloqueado / conflito não resolvido / score insuficiente)
3. Canal de escalada específico para o tema

**Mapa de escalada por tema (inicial — a ser expandido pelo Gestor do Corpus):**

| Tema | Canal de escalada |
|---|---|
| Frete de carga perigosa acima de 500 kg | Ramal 4500 — Gestão de Riscos |
| Frete reverso / custo de devolução por desistência | Comercial |
| Carga danificada em trânsito | sinistros@novatech.com.br |
| Seguro de carga | Comercial |
| Temas sem mapeamento específico | Supervisor de atendimento |

**Como testar:**  
Ao perguntar sobre frete de carga perigosa acima de 500 kg, a resposta deve conter: (1) "frete de carga perigosa acima de 500 kg", (2) "O PROC-043, que regula esse tema, está em revisão pelo Compliance e não está disponível no momento", (3) "Encaminhe ao ramal 4500 — Gestão de Riscos." A ausência de qualquer um dos três elementos é falha no teste.

---

### REQ-A03 — A classificação de cobertura bloqueada prevalece sobre qualquer chunk recuperado

**O que o sistema deve fazer:**  
Para temas na lista de cobertura bloqueada (REQ-F04), o assistente aciona escalada mesmo que chunks relacionados tenham sido recuperados com score alto. A verificação da lista de cobertura bloqueada é executada antes do uso dos chunks recuperados — não depois.

**Como testar:**  
Dado que "frete de carga perigosa acima de 500 kg" está na lista de cobertura bloqueada e o chunk do PROC-042 v2 mencionando "cargas perigosas seguem PROC-043" é recuperado com score 0.88, o assistente deve acionar escalada ao ramal 4500. NÃO deve usar o chunk do PROC-042 v2 para gerar resposta parcial sobre carga perigosa.

---

## Seção 4 — Requisitos de atualização do corpus

### REQ-U01 — Indexação de novos documentos: automática para normativos vigentes, validada para novos documentos sem histórico

**O que o sistema deve fazer:**  
A política de indexação difere por tipo de evento:

| Evento | Processo | SLA |
|---|---|---|
| Nova versão de documento existente com metadado `doc_substitui` preenchido | Indexação automática após validação de metadados obrigatórios (REQ-F02). Versão anterior marcada como `histórico` automaticamente (REQ-U02). | 48 horas úteis após publicação no SharePoint/Confluence |
| Novo documento sem histórico no corpus | Indexação aguarda aprovação do Gestor do Corpus. Pipeline notifica o Gestor ao detectar o documento. | Aprovação em até 3 dias úteis; indexação em até 48h após aprovação |
| Documento com `doc_tem_disposicao_transitoria = true` | Indexação automática; ambas as versões mantidas ativas (REQ-U04). Gestor do Corpus notificado. | 48 horas úteis |
| Atualização de planilha de referência mensal | Indexação automática; versão anterior desativada. | Até o 2º dia útil do mês |

**Por quê:**  
Indexação automática de novos documentos sem histórico representa risco: um documento com erro publicado acidentalmente seria disponibilizado ao atendente sem revisão. A distinção por tipo de evento equilibra velocidade e segurança.

**Como testar:**  
Dado que POL-001 v3.2 é publicada com metadado `doc_substitui: POL-001 v3.1`, o pipeline deve indexá-la automaticamente em até 48 horas úteis, sem intervenção do Gestor do Corpus. Dado que um documento novo `PROC-099` é publicado sem histórico no corpus, o pipeline deve notificar o Gestor do Corpus e aguardar aprovação antes de indexar.

---

### REQ-U02 — Nova versão rebaixa versão anterior para histórico, exceto quando há disposição transitória

**O que o sistema deve fazer:**  
Ao indexar uma nova versão de documento (identificada pelo metadado `doc_substitui`), o pipeline deve automaticamente alterar o `doc_status` da versão anterior de `vigente` para `histórico`. A versão histórica permanece acessível via interface de administração do Gestor do Corpus, mas é excluída das buscas do atendente.

**Exceção — disposição transitória:** Se a versão anterior tem `doc_tem_disposicao_transitoria = true`, o rebaixamento automático NÃO ocorre. O Gestor do Corpus deve decidir manualmente se e quando rebaixar a versão anterior, com base no prazo definido na disposição transitória do documento.

**Como testar:**  
Dado que POL-001 v3.2 é indexada com `doc_substitui: POL-001 v3.1` e POL-001 v3.1 tem `doc_tem_disposicao_transitoria = false`, o sistema deve automaticamente marcar POL-001 v3.1 como `histórico`. Ao perguntar sobre prazo de devolução, a resposta deve ser baseada na v3.2.

Dado que PROC-042 v3 é indexada com `doc_substitui: PROC-042 v2` e PROC-042 v2 tem `doc_tem_disposicao_transitoria = true`, o sistema NÃO deve rebaixar automaticamente o v2. Deve notificar o Gestor do Corpus para decisão manual.

---

### REQ-U03 — Documentos com atualização mensal têm alerta automático de vencimento e bloqueio de uso

**O que o sistema deve fazer:**  
Documentos com `doc_classificacao_ciclo = mensal` e `doc_validade_ate` preenchido devem acionar dois comportamentos automáticos na data de vencimento:

1. **Alerta ao Gestor do Corpus:** notificação automática no canal definido (Teams ou e-mail) com a lista de documentos vencidos.
2. **Bloqueio de uso para cálculo:** o documento vencido não pode ser usado como base para respostas que envolvam cálculo de valores. Pode continuar sendo usado para respostas sobre regras e procedimentos que não dependam de valores atualizados mensalmente.

**Como testar:**  
Dado que a tabela de frete base de janeiro venceu em 31/01, no dia 01/02 o Gestor do Corpus deve receber o alerta. Ao perguntar sobre valor de frete no dia 02/02 sem nova tabela indexada, o assistente deve usar a mensagem de tabela desatualizada (REQ-F05) e não deve usar a tabela de janeiro para calcular valores.

---

### REQ-U04 — Documentos com disposição transitória explícita preservam ambas as versões com escopo temporal definido

**O que o sistema deve fazer:**  
Quando um documento tem `doc_tem_disposicao_transitoria = true`, ambas as versões (nova e anterior) são mantidas com `doc_status = vigente_transitório`. O metadado `doc_escopo_temporal` define o intervalo de chamados para o qual cada versão é aplicável.

O sistema usa a data de abertura do chamado (obtida via REQ-C04 — campo obrigatório para perguntas de frete) para selecionar a versão correta antes de gerar a resposta.

**Configuração atual para PROC-042:**

| Documento | `doc_status` | `doc_escopo_temporal` |
|---|---|---|
| PROC-042 v1 | `vigente_transitório` | Chamados com data de abertura anterior a 01/12/2023 |
| PROC-042 v2 | `vigente_transitório` | Chamados com data de abertura a partir de 01/12/2023 |

**Como testar:**  
Dado que PROC-042 v1 e v2 estão indexados com escopos temporais definidos, ao perguntar sobre multiplicador para o Nordeste para chamado aberto em outubro de 2023, o assistente deve usar o PROC-042 v1 (multiplicador 1.4). Para chamado de fevereiro de 2024, deve usar o PROC-042 v2 (multiplicador 1.5).

---

## Seção 5 — Requisitos de rastreabilidade

### REQ-R01 — Toda resposta deve citar fonte com nome, versão, seção e data

**O que o sistema deve fazer:**  
Nenhuma resposta é entregue ao atendente sem citação completa. A citação deve incluir: nome do documento (`doc_nome`), versão (`doc_versao`), seção específica, e data de atualização (`doc_data_atualizacao`). A citação aparece no corpo da resposta, visível ao atendente sem ação adicional.

**Como testar:**  
Ao perguntar "qual o SLA de primeira resposta para cliente Gold em incidente crítico?", a resposta deve incluir: "30 minutos. *Fonte: SLA-2024 v2024.1, seção 2 — Tabela de SLAs — atualizado em 02/01/2024.*" A ausência de qualquer um dos elementos da citação é falha no teste.

---

### REQ-R02 — O trecho original do documento deve ser acessível via botão na interface

**O que o sistema deve fazer:**  
Toda resposta do assistente deve incluir um botão "Ver trecho original" que, ao ser acionado, exibe o chunk exato recuperado do documento com destaque no trecho específico que fundamentou a resposta. O texto exibido deve ser idêntico ao documento indexado — sem paráfrase ou edição.

Para respostas baseadas em múltiplos chunks, o botão deve exibir todos os chunks utilizados, identificados por documento e seção.

Para temas classificados como de alto risco (carga perigosa, incidentes críticos, cálculo de frete com valor acima de R$ 10.000), o trecho original deve ser exibido automaticamente junto com a resposta — sem necessidade de acionamento pelo atendente.

**Como testar:**  
Dado que o atendente recebeu resposta sobre prazo de devolução e clicou em "Ver trecho original", o sistema deve exibir o texto da POL-001 v3.1, seção 3.1, sem modificações. Dado que a resposta envolve cálculo de frete acima de R$ 10.000, o trecho deve aparecer automaticamente sem o atendente precisar clicar.

---

### REQ-R03 — Respostas baseadas em fontes informais devem ser sinalizadas visualmente

**O que o sistema deve fazer:**  
Quando uma resposta for gerada a partir de item do FAQ indexado como "fonte informal", o assistente deve exibir aviso visual em destaque imediatamente antes da resposta: "⚠️ Fonte não validada por Compliance. Confirme com o documento normativo antes de usar."

O aviso não impede a exibição da resposta. A presença do aviso e o conteúdo da resposta são exibidos juntos.

**Como testar:**  
Dado que o item 22 do FAQ (seguro de carga) está indexado como fonte informal, ao perguntar sobre percentual de seguro de carga padrão, a resposta deve exibir o aviso visual antes do conteúdo. A ausência do aviso é falha no teste.

---

### REQ-R04 — O sistema mantém log de consultas acessível ao Gestor do Corpus com SLA de disponibilidade definido

**O que o sistema deve fazer:**  
O pipeline mantém log de cada consulta contendo: timestamp, texto da pergunta, documentos recuperados com scores de similaridade, chunks utilizados na geração, e resposta final gerada.

**Acesso ao log:**

| Perfil | Nível de acesso | Finalidade |
|---|---|---|
| Gestor do Corpus | Acesso completo — todos os campos | Investigação de erros reportados, melhoria do corpus |
| Administrador do sistema | Acesso completo + export | Auditoria, conformidade |
| Atendente | Sem acesso direto | Usa o fluxo de feedback da Jornada para reportar erros |

**SLA de disponibilidade:** O log de uma consulta deve estar disponível para o Gestor do Corpus em até 2 horas após a consulta (não 24 horas como na v1.0 — reduzido para permitir investigação ágil de erros em chamados em aberto).

**Conexão com o fluxo de feedback:** Quando o atendente reporta erro via Teams (fluxo de feedback da Jornada), o sistema deve vincular automaticamente o reporte ao log da consulta correspondente, identificada pelo timestamp e ID do atendente.

**Como testar:**  
Após o atendente reportar erro às 14h, o Gestor do Corpus deve conseguir acessar o log da consulta reportada até as 16h do mesmo dia. O log deve identificar quais chunks foram usados e seus scores, permitindo ao Gestor determinar se o erro veio de documento desatualizado, conflito não detectado, ou falha de interpretação.

---

## Seção 6 — Requisitos de escopo de usuário e acesso

### REQ-S01 — Controle de acesso por perfil de usuário

**O que o sistema deve fazer:**  
O assistente deve reconhecer três perfis de usuário com permissões distintas sobre o corpus:

| Perfil | Acesso ao corpus | Acesso a fontes informais | Acesso a documentos históricos |
|---|---|---|---|
| Atendente | Documentos com `doc_status = vigente` ou `vigente_transitório` | Sim, com aviso visual (REQ-R03) | Não |
| Supervisor de atendimento | Idem atendente + documentos `pendente_classificação` (somente leitura) | Sim, com aviso visual | Sim, com indicação de status |
| Gestor do Corpus | Acesso completo a todos os status | Sim | Sim |

O perfil é determinado pelo login do usuário na integração com Microsoft 365 (Teams + Azure AD). Atendentes não devem conseguir acessar documentos históricos ou pendentes de classificação diretamente via assistente.

⚠️ **Decisão pendente NovaTech:** Definir quais cargos do time de atendimento correspondem a cada perfil e mapear para grupos do Azure AD.

**Como testar:**  
Dado que um atendente pergunta sobre um tema coberto apenas por documento com status `histórico`, o assistente deve responder com mensagem de ausência e escalada — não deve exibir o conteúdo do documento histórico.

---

### REQ-S02 — Escopo de idioma e variação regional

**O que o sistema deve fazer:**  
O corpus, as perguntas e as respostas do assistente operam exclusivamente em português brasileiro. O assistente não deve aceitar perguntas em outros idiomas ou gerar respostas em outros idiomas.

Para variação regional, os seguintes elementos do corpus têm escopo geográfico explícito e devem ser tratados com o multiplicador ou prazo correspondente à região informada na pergunta:
- Multiplicadores regionais do PROC-042 v2 (Sul, Sudeste, Centro-Oeste, Nordeste, Norte)
- Prazos de entrega por rota (quando disponíveis no corpus)
- Feriados regionais que afetam contagem de dias úteis de SLA: o sistema usa calendário nacional de feriados. Feriados estaduais ou municipais não são considerados no cálculo automático — o assistente deve informar essa limitação quando perguntado sobre datas próximas a feriados regionais.

**Como testar:**  
Ao perguntar "what is the delivery SLA for Gold clients?", o assistente deve responder em português: "Por favor, faça sua pergunta em português." — e não tentar responder em inglês. Ao perguntar sobre prazo para região Norte, o assistente deve usar o multiplicador 1.8 do PROC-042 v2 e o prazo padrão da rota + 3 dias úteis.

---

## Seção 7 — Requisitos de performance e disponibilidade

### REQ-P01 — Latência e volume de consultas

**O que o sistema deve fazer:**  
O assistente deve atender os seguintes requisitos de performance em condições normais de operação:

| Métrica | Valor alvo | Valor máximo aceitável |
|---|---|---|
| Latência de resposta (percentil 95) | Até 15 segundos | Até 30 segundos |
| Latência de resposta (percentil 99) | Até 30 segundos | Até 60 segundos |
| Consultas simultâneas suportadas | 45 (1 por atendente) | 60 (com degradação aceitável de latência) |
| Disponibilidade mensal | 99,0% | — |

⚠️ **Decisão pendente NovaTech:** Validar se os valores de latência acima são aceitáveis para o fluxo de atendimento. Com meta de busca < 2 minutos, uma latência de 30 segundos representa 25% do tempo disponível — o que pode ser aceitável ou não dependendo do tempo de digitação da pergunta e leitura da resposta pelo atendente.

**Por quê:**  
Com 320 chamados/dia e ~60% com consulta documental (192 consultas/dia), e considerando picos de demanda na proporção 3:1 em relação à média, o sistema deve suportar até ~24 consultas simultâneas em horário de pico. O valor de 45 simultâneas garante margem para crescimento.

**Como testar:**  
Dado cenário de carga com 45 consultas simultâneas, o tempo de resposta do percentil 95 deve ser inferior a 15 segundos. Dado 1 consulta isolada, o tempo de resposta deve ser inferior a 8 segundos.

---

### REQ-P02 — Comportamento em modo offline ou degradado

**O que o sistema deve fazer:**  
O assistente deve operar em três modos com comportamentos distintos:

**Modo normal:** Todos os requisitos desta spec aplicados integralmente.

**Modo degradado (latência alta / serviço parcialmente disponível):** O assistente exibe banner de aviso: "O assistente está operando com desempenho reduzido. Respostas podem demorar mais que o usual." Funcionalidades mantidas: busca e resposta. Funcionalidades suspensas: exibição de trecho original (REQ-R02) e log em tempo real (REQ-R04).

**Modo offline (serviço indisponível):** O assistente exibe mensagem clara: "O assistente está temporariamente indisponível. Use os documentos diretamente:" seguida de links diretos para SharePoint (documentos normativos) e Confluence (wiki interna). O Gestor do Corpus e a equipe técnica são notificados automaticamente. O SLA de restauração do serviço deve ser definido pelo time técnico na especificação de infraestrutura.

**Como testar:**  
Dado que o serviço de busca vetorial está indisponível, o assistente deve exibir a mensagem de modo offline com links para as fontes diretas. NÃO deve tentar responder com conhecimento geral do modelo nem exibir erro técnico sem orientação ao atendente.

---

## Seção 8 — Resumo executivo dos requisitos

### Tabela completa de requisitos

| ID | Área | Requisito resumido | Bloqueia go-live? | Novo na v2? |
|---|---|---|---|---|
| REQ-F01 | Fontes | Campo `doc_status` obrigatório; curadoria inicial dos ~800 docs | Sim | Atualizado |
| REQ-F02 | Fontes | 10 metadados obrigatórios por documento | Sim | Atualizado |
| REQ-F03 | Fontes | FAQ indexado por item, com critério de elegibilidade | Sim | Atualizado |
| REQ-F04 | Fontes | Lista de cobertura bloqueada com mapa de escalada | Sim | Atualizado |
| REQ-F05 | Fontes | Planilhas mensais com vencimento e bloqueio automático | Não | Mantido |
| REQ-C01 | Contradições | Detecção por lista explícita + heurística por `doc_codigo` | Sim | Reescrito |
| REQ-C02 | Contradições | Tabela de fontes autoritativas com governança definida | Sim | Reescrito |
| REQ-C03 | Contradições | Conflito sem resolução → escalada obrigatória | Sim | Mantido |
| REQ-C04 | Contradições | Política de esclarecimento com campos obrigatórios por tipo | Sim | Novo |
| REQ-A01 | Ausência | Critério de confiança: threshold 0.75 + lista de condições | Sim | Reescrito |
| REQ-A02 | Ausência | Mensagem de ausência com 3 elementos obrigatórios + mapa de escalada | Não | Atualizado |
| REQ-A03 | Ausência | Cobertura bloqueada verificada antes dos chunks | Sim | Mantido |
| REQ-U01 | Atualização | Indexação automática vs. validada por tipo de evento | Não | Reescrito |
| REQ-U02 | Atualização | Nova versão rebaixa anterior; exceto disposição transitória | Sim | Atualizado |
| REQ-U03 | Atualização | Vencimento mensal com alerta e bloqueio parcial de uso | Não | Atualizado |
| REQ-U04 | Atualização | Disposição transitória preserva ambas as versões com escopo temporal | Sim | Atualizado |
| REQ-R01 | Rastreabilidade | Citação com nome, versão, seção e data em toda resposta | Sim | Mantido |
| REQ-R02 | Rastreabilidade | Botão "Ver trecho original"; automático para temas de alto risco | Não | Atualizado |
| REQ-R03 | Rastreabilidade | Fontes informais com aviso visual obrigatório | Sim | Mantido |
| REQ-R04 | Rastreabilidade | Log com acesso por perfil e SLA de 2h para o Gestor | Não | Reescrito |
| REQ-S01 | Escopo | Controle de acesso por 3 perfis via Azure AD | Sim | Novo |
| REQ-S02 | Escopo | Português brasileiro; feriados nacionais; regiões do PROC-042 | Não | Novo |
| REQ-P01 | Performance | Latência P95 ≤ 15s; 45 consultas simultâneas; 99,0% disponibilidade | Não | Novo |
| REQ-P02 | Performance | Modo offline com links diretos às fontes; modo degradado com banner | Não | Novo |

### Decisões pendentes de validação pela NovaTech antes do go-live

| # | Decisão | Responsável sugerido | Impacto se não decidida |
|---|---|---|---|
| D-01 | Nomear o Gestor do Corpus (papel) | Diretoria de Operações | Tabela de fontes autoritativas sem dono; processo de curadoria sem responsável |
| D-02 | Provisionar campo `doc_status` no SharePoint e Confluence | TI + Operações | Nenhum documento pode ser indexado sem esse campo |
| D-03 | Concluir curadoria inicial dos ~800 documentos existentes | Gestor do Corpus + áreas responsáveis | Corpus inicial vazio ou com documentos de status desconhecido |
| D-04 | Validar tabela de fontes autoritativas com Comercial, Operações e Compliance | Gestor do Corpus | Conflitos não resolvidos geram escalada em vez de resposta |
| D-05 | Validar elegibilidade dos 47 itens do FAQ com Compliance | Gestor do Corpus + Compliance | FAQ não pode ser indexado (nem como fonte informal) |
| D-06 | Definir perfis de usuário e mapear para grupos Azure AD | TI + RH | Controle de acesso (REQ-S01) não pode ser implementado |
| D-07 | Validar SLAs de latência do REQ-P01 com time de atendimento | Gestor de Produto + Coord. de Atendimento | Requisito de performance pode estar sub ou superdimensionado |
| D-08 | Validar campos obrigatórios por tipo de pergunta (REQ-C04) | Gestor do Corpus + Coord. de Atendimento | Política de esclarecimento pode gerar atrito desnecessário ou lacunas |

### Pré-condições para go-live

1. Todas as decisões D-01 a D-08 tomadas e documentadas.
2. Campo `doc_status` provisionado e curadoria inicial concluída (D-02 e D-03).
3. Tabela de fontes autoritativas validada e configurada no sistema (D-04).
4. Lista de cobertura bloqueada configurada no sistema (mínimo: PROC-043, PROC-088, frete reverso pré-2023).
5. Todos os requisitos marcados como "bloqueia go-live" com casos de teste aprovados.
6. Treinamento do time de atendimento: citações de fonte, botão de trecho original, fluxo de feedback, comportamento esperado nos modos degradado e offline.
7. Threshold de confiança (REQ-A01, valor inicial 0.75) configurado e monitoramento ativo para calibração nos primeiros 30 dias.

---

*Versão 2.0 — produzida na fase de discovery do projeto NovaTech × DB1*  
*Substitui: spec-requisitos-rag-novatech-v1.0.md*  
*Complementa: jornada-atendente-novatech.md, cruzamento-inconsistencias-faq-novatech.md*  
*Próximo passo: revisão das decisões pendentes (D-01 a D-08) com as áreas responsáveis*
