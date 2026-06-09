# Especificação de Requisitos de Produto — Pipeline RAG
## Assistente de Atendimento NovaTech

> **Versão:** 1.0 — rascunho para revisão  
> **Projeto:** NovaTech × DB1 — Assistente de IA para Atendimento  
> **Escopo:** Requisitos funcionais do pipeline RAG sob a perspectiva de produto  
> **Base:** Discovery documental (Etapas 1–3) + Jornada do atendente (Etapa 4) + Anexo A  
> **Não cobre:** Arquitetura técnica, escolha de modelos, infraestrutura Azure

---

## Como ler este documento

Esta especificação descreve **o que o sistema deve fazer**, não como deve ser implementado tecnicamente. Cada requisito é formulado de forma que um profissional de QA consiga escrever um caso de teste sem pedir esclarecimentos adicionais.

Os exemplos concretos ao longo do documento são extraídos diretamente da documentação real da NovaTech (Anexo A) e das inconsistências identificadas no discovery. Eles existem para tornar os requisitos verificáveis — não são exaustivos.

**Estrutura de cada requisito:**
- **O que o sistema deve fazer** — comportamento esperado
- **Por quê** — motivação rastreada ao discovery ou à jornada
- **Como testar** — critério de aceitação verificável
- **Exemplo concreto** — caso real da documentação NovaTech

---

## Seção 1 — Fontes de dados: o que deve e o que não deve ser indexado

### REQ-F01 — Somente documentos com status de vigência confirmado devem ser indexados como fonte autoritativa

**O que o sistema deve fazer:**  
Antes de indexar qualquer documento, o pipeline deve verificar se o documento possui um status de vigência definido. Documentos sem status explícito (nem "vigente" nem "obsoleto") não devem ser indexados como fonte autoritativa. Devem permanecer acessíveis para auditoria, mas sinalizados como "pendente de classificação" e excluídos das buscas respondidas ao atendente.

**Por quê:**  
O PROC-042 v1 e o PROC-042 v2 coexistem no SharePoint sem que nenhum dos dois possua marcação formal de vigência ou obsolescência. Ambos têm o mesmo nome de tema e fórmula de cálculo — mas fatores de peso e multiplicadores regionais diferentes. Se os dois forem indexados sem distinção de hierarquia, o mecanismo de busca pode recuperar ambos para a mesma pergunta e o assistente gerará respostas contraditórias ou calculará valores errados.

**Como testar:**  
Dado que PROC-042 v1 e PROC-042 v2 estão no corpus sem status de vigência marcado, ao perguntar "qual o fator de peso para carga de 2.000 kg?", o assistente NÃO deve retornar um valor calculado. Deve retornar uma mensagem de escalada indicando que há conflito documental não resolvido nesse tema.

**Exemplo concreto:**  
PROC-042 v1 (mar/2023): fator de peso 1.2 para cargas de 1.001–3.000 kg.  
PROC-042 v2 (nov/2023): fator de peso 1.15 para a mesma faixa.  
Diferença de 4,2% no valor final do frete — erro financeiro recorrente se o sistema escolher a versão errada.

---

### REQ-F02 — Documentos com data de vigência explícita devem ser indexados com metadado de vigência obrigatório

**O que o sistema deve fazer:**  
Todo documento indexado deve carregar, no mínimo, os seguintes metadados: nome do documento, versão, data de emissão/atualização, área responsável, e classificação (normativo / informativo / informal). Esses metadados devem ser usados pelo sistema na geração da resposta e exibidos ao atendente junto com a citação de fonte.

**Por quê:**  
O atendente precisa saber não apenas qual documento gerou a resposta, mas se esse documento é recente o suficiente para ser confiável. A POL-001 existia em versão truncada (sem conteúdo real) e foi substituída pela v3.1 de 15/01/2024 — sem os metadados de versão, o sistema não consegue distinguir qual das duas versões está indexada.

**Como testar:**  
Dado que POL-001 v3.1 está indexada, ao perguntar "qual o prazo para solicitar devolução?", a resposta deve incluir: nome do documento (POL-001), versão (3.1), data de atualização (15/01/2024) e seção exata (3.1 — Prazo geral). A ausência de qualquer um desses campos deve ser tratada como falha no teste.

**Exemplo concreto:**  
Resposta esperada: "O prazo para devolução é de 7 dias úteis após a confirmação de recebimento no sistema de tracking. *Fonte: POL-001 v3.1, seção 3.1 — atualizada em 15/01/2024 pela Diretoria de Operações.*"

---

### REQ-F03 — O FAQ informal não deve ser indexado como fonte normativa

**O que o sistema deve fazer:**  
O arquivo FAQ-Atendimento não deve ser indexado junto ao corpus de documentos normativos. Se for indexado para fins de cobertura de temas não documentados formalmente, deve receber metadado de classificação "fonte informal — não validada" e o assistente deve sinalizar essa classificação em toda resposta gerada a partir dele.

**Por quê:**  
O FAQ foi criado organicamente pelo time ao longo de 2 anos sem validação por Compliance ou Operações. Ele contém ao menos um erro factual crítico (limiar de desconto: 10 fretes no FAQ vs 8 fretes no PROC-042 v2) e orientações sobre temas sem procedimento normativo correspondente (seguro de carga, carga danificada, frete expresso com carga perigosa). Indexado como fonte de mesmo peso que documentos normativos, contamina respostas com informações não validadas.

**Como testar:**  
Dado que o FAQ-Atendimento está no corpus com metadado "fonte informal", ao perguntar "a partir de quantos fretes especiais por mês o cliente tem direito a desconto automático?", o sistema deve retornar a resposta baseada no PROC-042 v2 (8 fretes/mês), não no FAQ (10 fretes/mês). Se o PROC-042 v2 não estiver disponível, deve escalar ao Comercial — não usar o FAQ como fallback para esse tema.

**Exemplo concreto:**  
FAQ item 45: "Para clientes com mais de 10 fretes especiais por mês, existe desconto automático."  
PROC-042 v2, seção 4: "A partir de 8 fretes especiais/mês, desconto de 5%; acima de 15 fretes/mês, desconto de 10%."  
Usar o FAQ aqui nega desconto a clientes com 8 ou 9 fretes/mês — impacto financeiro direto e recorrente.

---

### REQ-F04 — Documentos referenciados mas ausentes do corpus devem gerar alerta de cobertura bloqueada

**O que o sistema deve fazer:**  
Quando um documento indexado faz referência a outro documento que não está no corpus (ex: "ver PROC-043" ou "consultar PROC-088"), o sistema deve registrar esse documento ausente como um tema com cobertura bloqueada. Perguntas sobre esses temas devem acionar escalada, não tentativa de resposta.

**Por quê:**  
O PROC-042 v2 delega o cálculo de frete para cargas perigosas acima de 500 kg ao PROC-043, que está em revisão pelo Compliance e ausente do corpus. A POL-001 v3.1 menciona o PROC-088 (Interceptação de Carga) para mercadorias em trânsito, também ausente. Se o assistente tentar responder sobre esses temas sem o documento de referência, gerará alucinação.

**Como testar:**  
Dado que PROC-043 não está indexado, ao perguntar "qual o valor do frete para carga perigosa de 800 kg para o Norte?", o assistente deve responder: "Esse tema é regulado pelo PROC-043, que está em revisão pelo Compliance e não está disponível no momento. Encaminhe ao ramal 4500 — Gestão de Riscos." O assistente NÃO deve tentar calcular um valor.

---

### REQ-F05 — Planilhas de referência com atualização mensal devem ser tratadas como fonte com validade limitada

**O que o sistema deve fazer:**  
A tabela mensal de fretes base (referenciada no PROC-042 v1 e v2 como `frete-base-AAAAMM.xlsx`) deve ser indexada com data de validade explícita correspondente ao mês de vigência. O sistema deve verificar se a planilha indexada corresponde ao mês corrente. Se estiver desatualizada, o assistente deve sinalizar ao atendente que o valor base pode não ser o vigente.

**Por quê:**  
O valor base do frete é atualizado mensalmente. Usar uma planilha do mês anterior para calcular frete pode gerar valor incorreto para o cliente. O pipeline precisa de mecanismo para identificar e sinalizar essa condição.

**Como testar:**  
Dado que a planilha de frete base indexada é do mês anterior ao mês corrente, ao perguntar sobre valor de frete, o assistente deve incluir o aviso: "O valor base utilizado neste cálculo é o de [mês/ano]. Confirme com o sistema se a tabela foi atualizada para o mês corrente."

---

## Seção 2 — Comportamento com documentos contraditórios

### REQ-C01 — O sistema deve detectar conflito entre documentos antes de gerar resposta

**O que o sistema deve fazer:**  
Quando o mecanismo de busca recuperar chunks de dois ou mais documentos diferentes que respondem à mesma pergunta com valores ou regras distintos, o sistema não deve escolher um deles arbitrariamente. Deve identificar o conflito, apresentar as duas versões ao atendente com indicação de fonte para cada uma, e recomendar escalada para a área responsável por definir qual versão prevalece.

**Por quê:**  
O pipeline RAG recupera chunks por similaridade semântica — se dois documentos têm conteúdo parecido (mesma fórmula, faixas de peso idênticas, valores diferentes), ambos podem ser recuperados para a mesma pergunta. Deixar o LLM escolher qual usar é delegar uma decisão de negócio para o modelo — o modelo não tem como saber qual versão é a vigente.

**Como testar:**  
Dado que PROC-042 v1 e v2 estão ambos indexados (hipótese de transição incompleta), ao perguntar "qual o prazo adicional para frete especial?", o assistente deve apresentar as duas versões:
- "PROC-042 v1 (mar/2023): prazo padrão + 2 dias úteis"
- "PROC-042 v2 (nov/2023): prazo padrão + 3 dias úteis"

E deve incluir: "Há duas versões deste procedimento no sistema. Confirme com o Comercial qual versão aplica ao contrato deste cliente."  
O assistente NÃO deve escolher uma das versões e apresentá-la como única resposta.

---

### REQ-C02 — A política de fonte autoritativa por tema deve ser configurável e aplicada antes da geração da resposta

**O que o sistema deve fazer:**  
O pipeline deve suportar a configuração de uma tabela de fontes autoritativas por tema: para cada tema coberto pelo corpus, define-se qual documento é a fonte de verdade. Quando há conflito, o sistema usa essa tabela para priorizar o documento correto — sem precisar que o LLM decida.

**Tabela inicial de fontes autoritativas (a ser validada com as áreas responsáveis):**

| Tema | Fonte autoritativa | Área responsável pela manutenção |
|---|---|---|
| Classificação de tiers e SLA | SLA-2024 v2024.1 | Comercial + Operações |
| Cálculo de frete especial (chamados a partir de 01/12/2023) | PROC-042 v2 | Comercial |
| Cálculo de frete especial (chamados anteriores a 01/12/2023) | PROC-042 v1 | Comercial |
| Prazo e processo de devolução | POL-001 v3.1 | Operações |
| Incidentes críticos e penalidades de SLA | SLA-2024 v2024.1 | Comercial + Operações |
| Desconto por volume de frete | PROC-042 v2, seção 4 | Comercial |

**Por quê:**  
A regra de prioridade por data de emissão não funciona isoladamente — o PROC-042 v1 deve continuar sendo a fonte correta para chamados anteriores a 01/12/2023, mesmo sendo o documento mais antigo. A política de fonte autoritativa captura essa nuance que uma regra de "usar o mais recente" não capturaria.

**Como testar:**  
Dado que a tabela de fontes autoritativas está configurada com PROC-042 v2 como fonte para chamados a partir de 01/12/2023, ao perguntar sobre fator de peso para uma carga de 2.000 kg sem informar a data do chamado, o sistema deve solicitar a data antes de gerar a resposta. Com data informada após 01/12/2023, deve usar os fatores da v2 (1.15). Com data anterior, deve usar os fatores da v1 (1.2).

---

### REQ-C03 — Conflitos sem resolução na tabela de fontes devem acionar escalada obrigatória, nunca resposta estimada

**O que o sistema deve fazer:**  
Se o tema da pergunta tem conflito documental e não há fonte autoritativa definida na tabela, o assistente deve recusar gerar uma resposta com valor ou regra específica. Deve apresentar os documentos conflitantes disponíveis, explicar que há conflito sem resolução formal, e orientar escalada.

**Por quê:**  
Uma resposta errada apresentada com confiança é mais prejudicial que nenhuma resposta. O atendente que recebe um prazo ou valor incorreto do assistente pode repassá-lo ao cliente como informação oficial da NovaTech — gerando impacto contratual, financeiro ou de imagem.

**Como testar:**  
Dado que não há fonte autoritativa definida para "custo do frete reverso por desistência do cliente em contratos pré-2023", ao perguntar sobre esse tema, o assistente deve responder:  
"Há ambiguidade neste tema entre documentos. A POL-001 v3.1 define que o frete reverso usa 'os mesmos multiplicadores do frete original', mas não especifica qual versão do PROC-042 aplicar para contratos anteriores a 01/12/2023. Encaminhe ao Comercial para definição."  
O assistente NÃO deve apresentar um valor calculado.

---

## Seção 3 — Comportamento quando não há resposta na base

### REQ-A01 — Quando não há resposta no corpus, o assistente deve declarar explicitamente a ausência — nunca usar conhecimento geral

**O que o sistema deve fazer:**  
Se o mecanismo de busca não recuperar chunks relevantes para a pergunta do atendente, o assistente deve informar que o tema não está coberto pelo corpus atual e orientar o canal de escalada correspondente. O assistente NÃO deve tentar responder com base em conhecimento geral do modelo de linguagem.

**Por quê:**  
O modelo de linguagem tem conhecimento genérico sobre logística, fretes e políticas comerciais — mas esse conhecimento não é a política da NovaTech. Uma resposta gerada com conhecimento geral pode parecer plausível e ser usada pelo atendente sem verificação, criando inconsistência com o contrato do cliente.

**Como testar:**  
Dado que não há documento sobre frete padrão (abaixo de 500 kg) no corpus, ao perguntar "qual o prazo de entrega para uma carga de 200 kg de São Paulo para o Norte?", o assistente deve responder:  
"Não encontrei informação sobre frete padrão no corpus atual. Esse tema pode estar em documentos que ainda não foram indexados. Consulte o supervisor ou o sistema de cotação."  
O assistente NÃO deve estimar um prazo com base em conhecimento geral sobre rotas de transporte.

---

### REQ-A02 — A mensagem de ausência de resposta deve ser específica ao tema, não genérica

**O que o sistema deve fazer:**  
A mensagem de ausência de resposta deve identificar o tema da pergunta e, quando possível, indicar por que não há cobertura (documento não indexado, tema bloqueado, conflito não resolvido) e qual o canal de escalada específico para aquele tema.

**Por quê:**  
Uma mensagem genérica como "não encontrei informação" não ajuda o atendente a saber o que fazer a seguir. O atendente precisa saber para onde escalar — e o canal certo varia por tema (ramal 4500 para carga perigosa, Comercial para desconto e frete reverso, sinistros@novatech.com.br para carga danificada).

**Como testar:**  
Dado que não há cobertura para frete de carga perigosa acima de 500 kg (PROC-043 ausente), ao perguntar sobre esse tema, a resposta deve:  
1. Identificar o tema: "frete de carga perigosa acima de 500 kg"  
2. Explicar o motivo da ausência: "O PROC-043, que regula esse tema, está em revisão pelo Compliance"  
3. Indicar o canal: "Encaminhe ao ramal 4500 — Gestão de Riscos"  

A mensagem NÃO deve ser: "Não encontrei informação sobre isso."

---

### REQ-A03 — O assistente não deve responder perguntas sobre temas com cobertura intencionalmente bloqueada, mesmo que tenha informação parcial no corpus

**O que o sistema deve fazer:**  
Para temas classificados como "cobertura bloqueada" (ver REQ-F04), o assistente deve acionar escalada mesmo que existam chunks relacionados recuperados pelo mecanismo de busca. A classificação de bloqueio prevalece sobre a recuperação de chunks.

**Por quê:**  
O PROC-042 v2 menciona cargas perigosas e as delega ao PROC-043. Esse trecho pode ser recuperado pela busca semântica quando o atendente pergunta sobre carga perigosa. Mas o trecho do PROC-042 v2 não é suficiente para responder — ele apenas redireciona para um documento que não existe no corpus. Permitir que o assistente use esse chunk parcial como base de resposta gera uma resposta incompleta com aparência de confiabilidade.

**Como testar:**  
Dado que PROC-042 v2 está indexado e PROC-043 está classificado como cobertura bloqueada, ao perguntar "qual o custo de frete para 600 kg de carga perigosa para o Nordeste?", o assistente deve acionar escalada para o ramal 4500. NÃO deve calcular o frete usando os multiplicadores do PROC-042 v2 aplicados a esse caso, mesmo que o chunk do PROC-042 v2 seja recuperado pela busca.

---

## Seção 4 — Requisitos de atualização do corpus

### REQ-U01 — Novos documentos normativos publicados pelas áreas responsáveis devem ser indexados em até 48 horas úteis após publicação

**O que o sistema deve fazer:**  
O pipeline deve monitorar as fontes de documentos (SharePoint NovaTech e Confluence) e detectar automaticamente novos documentos ou novas versões de documentos existentes. Ao detectar uma publicação nova, deve iniciar o processo de indexação e tornar o documento disponível para busca em até 48 horas úteis.

**Por quê:**  
A NovaTech tem três áreas que atualizam documentação mensalmente (Operações, Compliance, Comercial) sem processo unificado de revisão. O ciclo mensal de atualizações significa que documentos novos chegam com frequência. Um lag de indexação superior a 48 horas úteis significa que o assistente pode responder com informação desatualizada durante esse período — especialmente crítico para tabelas de frete base, que mudam todo mês.

**Como testar:**  
Dado que POL-001 v3.2 é publicada no SharePoint às 9h de uma segunda-feira, o documento deve estar disponível para busca pelo assistente até as 9h da quarta-feira da mesma semana (48 horas úteis). Após esse prazo, ao perguntar sobre o tema coberto pela v3.2, o assistente deve retornar a resposta baseada na v3.2, não na v3.1.

---

### REQ-U02 — Quando uma nova versão de documento é indexada, a versão anterior deve ser automaticamente rebaixada para status "histórico"

**O que o sistema deve fazer:**  
O pipeline deve identificar quando um documento novo é uma nova versão de um documento já indexado (por nome de documento, código ou metadado de "substitui"). Ao indexar a nova versão, deve alterar o status da versão anterior para "histórico" — mantendo-a acessível para auditoria, mas excluindo-a das buscas respondidas ao atendente.

**Por quê:**  
O problema do PROC-042 v1 coexistindo com o v2 é exatamente o que esse requisito previne no futuro. A versão histórica precisa ser preservada (chamados antigos podem precisar ser verificados com base na regra vigente na época), mas não pode contaminar as respostas para novos chamados.

**Exceção:** documentos com disposição transitória explícita (como o PROC-042 v2, que define que chamados anteriores a 01/12/2023 usam a v1) devem manter ambas as versões ativas com escopo de aplicação definido por data.

**Como testar:**  
Dado que POL-001 v3.2 é publicada e indexada com metadado "substitui POL-001 v3.1", o sistema deve automaticamente marcar POL-001 v3.1 como "histórico". Após a indexação da v3.2, ao perguntar sobre prazo de devolução, a resposta deve ser baseada na v3.2, não na v3.1. POL-001 v3.1 deve continuar acessível via busca de auditoria com filtro explícito de "incluir histórico".

---

### REQ-U03 — Documentos com atualização mensal (planilhas de referência) devem ter alerta automático de vencimento

**O que o sistema deve fazer:**  
Para documentos classificados com ciclo de atualização mensal (ex: tabela de fretes base), o sistema deve gerar alerta automático no dia 1 de cada mês se a versão indexada não tiver sido atualizada. O alerta deve ser enviado à equipe responsável pela gestão do corpus e deve bloquear o uso dessa fonte pelo assistente até que a versão atual seja indexada.

**Por quê:**  
A tabela de fretes base é referenciada pelos dois documentos PROC-042 como base do cálculo de frete. Ela é atualizada mensalmente. Usar uma tabela desatualizada para calcular frete gera valor incorreto para o cliente — o assistente não tem como saber se o valor está desatualizado sem esse controle.

**Como testar:**  
Dado que a tabela de frete base de janeiro não foi indexada até o dia 5 de fevereiro, o sistema deve: (a) enviar alerta à equipe gestora do corpus, e (b) ao receber pergunta sobre cálculo de frete, incluir aviso: "A tabela de fretes base pode estar desatualizada. Confirme com o sistema antes de comunicar o valor ao cliente."

---

### REQ-U04 — Atualizações em documentos com disposição transitória devem preservar ambas as versões com escopo de aplicação definido

**O que o sistema deve fazer:**  
Quando um documento possui disposição transitória com data de corte (como o PROC-042 v2, que define regras diferentes para chamados antes e depois de 01/12/2023), o sistema deve manter ambas as versões indexadas com o escopo de aplicação correto: versão anterior para chamados com data anterior ao corte, versão nova para chamados a partir do corte.

**Por quê:**  
Chamados de clientes com operações iniciadas antes de 01/12/2023 ainda podem estar em processamento. Se o sistema indexar apenas o PROC-042 v2 e descartar o v1, não conseguirá responder corretamente sobre esses chamados históricos.

**Como testar:**  
Dado que PROC-042 v1 e v2 estão ambos indexados com escopo de aplicação por data, ao perguntar "qual o multiplicador para o Nordeste para um chamado aberto em outubro de 2023?", o assistente deve responder com o multiplicador do PROC-042 v1 (1.4) — não do v2 (1.5). Para um chamado de fevereiro de 2024, deve usar o v2 (1.5).

---

## Seção 5 — Requisitos de rastreabilidade

### REQ-R01 — Toda resposta gerada pelo assistente deve citar a fonte com nome, versão, seção e data

**O que o sistema deve fazer:**  
Nenhuma resposta do assistente deve ser entregue ao atendente sem a citação completa da fonte que a fundamenta. A citação deve incluir: nome do documento, versão, seção específica (não apenas o documento inteiro) e data de atualização. A citação deve aparecer no corpo da resposta, não apenas em metadados invisíveis.

**Por quê:**  
O atendente precisa poder verificar a resposta na fonte original quando necessário — especialmente em casos de divergência com o que o cliente apresenta. Sem a citação de seção específica, o atendente teria que ler o documento inteiro para encontrar o trecho, anulando o ganho de tempo que o assistente deveria trazer.

**Como testar:**  
Ao perguntar "qual o SLA de primeira resposta para cliente Gold em incidente crítico?", a resposta deve incluir:  
- Resposta: "30 minutos"  
- Citação: "*Fonte: SLA-2024 v2024.1, seção 2 — Tabela de SLAs, linha 'Tempo de primeira resposta (incidentes críticos)' — atualizado em 02/01/2024.*"  

A ausência da citação de seção específica deve ser tratada como falha no teste.

---

### REQ-R02 — O assistente deve exibir o trecho relevante do documento junto com a resposta, quando solicitado

**O que o sistema deve fazer:**  
O atendente deve ter a opção de solicitar o trecho exato do documento que fundamentou a resposta. Quando acionado, o sistema deve exibir o chunk recuperado com destaque no trecho específico que gerou a resposta, sem alterar o texto original do documento.

**Por quê:**  
Em situações de contestação pelo cliente ou de dúvida do atendente, ter acesso ao trecho original — não apenas à resposta parafraseada — é necessário para que o atendente possa verificar a interpretação do assistente e, se necessário, escalar com evidência documental.

**Como testar:**  
Dado que o atendente recebeu a resposta sobre prazo de devolução e solicitou "mostrar trecho", o sistema deve exibir o texto original da POL-001 v3.1, seção 3.1, sem modificações. O trecho deve coincidir exatamente com o documento indexado — nenhuma palavra deve ser alterada ou parafraseada.

---

### REQ-R03 — Respostas baseadas em fontes informais devem ser sinalizadas visualmente como tal

**O que o sistema deve fazer:**  
Quando uma resposta for gerada a partir de uma fonte classificada como "informal" (ex: FAQ-Atendimento), o assistente deve exibir uma marcação visual explícita — por exemplo, um aviso em destaque: "⚠️ Esta resposta é baseada em fonte não validada por Compliance. Confirme com o documento normativo antes de usar."

**Por quê:**  
O atendente não necessariamente sabe distinguir um documento normativo de um documento informal só pelo nome. Sem marcação visual, respostas do FAQ informal e respostas da POL-001 normativa chegam ao atendente com aparência idêntica de confiabilidade — e o atendente pode usar ambas sem questionar.

**Como testar:**  
Dado que o FAQ está indexado com classificação "informal" e o atendente pergunta sobre seguro de carga (tema coberto apenas pelo FAQ item 22), a resposta deve incluir o aviso visual de fonte não validada. A ausência do aviso deve ser tratada como falha no teste. A presença do aviso não deve impedir a exibição da resposta — deve complementá-la.

---

### REQ-R04 — O sistema deve registrar, para cada consulta, os documentos recuperados e os chunks utilizados na geração da resposta

**O que o sistema deve fazer:**  
O pipeline deve manter um log de cada consulta realizada, contendo: timestamp da consulta, texto da pergunta, documentos recuperados (com scores de similaridade), chunks utilizados na geração da resposta, e resposta final gerada. Esse log deve ser acessível à equipe gestora do corpus para fins de auditoria, detecção de erros e melhoria contínua.

**Por quê:**  
O fluxo de feedback (definido na Jornada do Atendente) depende da capacidade de investigar por que o assistente gerou uma determinada resposta. Sem o log de chunks utilizados, a equipe responsável não consegue identificar se o erro veio de um documento desatualizado, de um conflito não detectado, ou de uma falha de interpretação do modelo.

**Como testar:**  
Após o atendente reportar uma resposta incorreta sobre o prazo de frete especial, a equipe gestora deve conseguir acessar o log da consulta correspondente e identificar: quais documentos foram recuperados, quais chunks foram usados, e qual era o score de similaridade de cada chunk. O log deve estar disponível em até 24 horas após a consulta.

---

## Seção 6 — Resumo executivo dos requisitos

### Tabela de requisitos por área

| ID | Área | Requisito resumido | Bloqueia go-live? |
|---|---|---|---|
| REQ-F01 | Fontes | Indexar apenas documentos com vigência confirmada | Sim |
| REQ-F02 | Fontes | Metadados obrigatórios em todo documento indexado | Sim |
| REQ-F03 | Fontes | FAQ não indexado como fonte normativa | Sim |
| REQ-F04 | Fontes | Documentos ausentes geram cobertura bloqueada | Sim |
| REQ-F05 | Fontes | Planilhas mensais com data de validade e alerta | Não |
| REQ-C01 | Contradições | Detectar conflito antes de gerar resposta | Sim |
| REQ-C02 | Contradições | Tabela de fontes autoritativas por tema | Sim |
| REQ-C03 | Contradições | Conflito sem resolução → escalada obrigatória | Sim |
| REQ-A01 | Ausência | Declarar ausência — nunca usar conhecimento geral | Sim |
| REQ-A02 | Ausência | Mensagem de ausência específica ao tema | Não |
| REQ-A03 | Ausência | Cobertura bloqueada prevalece sobre chunks recuperados | Sim |
| REQ-U01 | Atualização | Novos documentos indexados em até 48h úteis | Não |
| REQ-U02 | Atualização | Nova versão rebaixa versão anterior para "histórico" | Sim |
| REQ-U03 | Atualização | Alerta automático para planilhas mensais vencidas | Não |
| REQ-U04 | Atualização | Disposições transitórias preservam ambas as versões | Sim |
| REQ-R01 | Rastreabilidade | Toda resposta cita fonte com nome, versão e seção | Sim |
| REQ-R02 | Rastreabilidade | Trecho original disponível sob demanda | Não |
| REQ-R03 | Rastreabilidade | Fontes informais sinalizadas visualmente | Sim |
| REQ-R04 | Rastreabilidade | Log de consultas para auditoria e feedback | Não |

### Pré-condições para go-live

Antes de o assistente ser disponibilizado para o time de atendimento, as seguintes condições devem ser satisfeitas:

1. **Curadoria documental concluída:** PROC-042 v1 e v2 com vigência formalmente definida; POL-001 v3.1 confirmada como única versão ativa; FAQ classificado como "fonte informal".
2. **Tabela de fontes autoritativas validada** pelas áreas Comercial, Operações e Compliance.
3. **Lista de temas com cobertura bloqueada** publicada e configurada no sistema (mínimo: frete de carga perigosa acima de 500 kg; custo de frete reverso em contratos pré-2023).
4. **Todos os requisitos marcados como "bloqueia go-live"** verificados com casos de teste aprovados.
5. **Treinamento do time de atendimento** sobre como interpretar citações de fonte, como usar o botão de trecho original, e como acionar o fluxo de feedback.

---

*Documento produzido na fase de discovery do projeto NovaTech × DB1*  
*Complementa: jornada-atendente-novatech.md e cruzamento-inconsistencias-faq-novatech.md*  
*Próximo passo: revisão com as áreas Comercial, Operações e Compliance para validação da tabela de fontes autoritativas e da lista de temas bloqueados*
