# Cruzamento de Inconsistências × Práticas Informais do FAQ — NovaTech

> Análise cruzada entre as inconsistências catalogadas (IC-01 a IC-06) e os itens do FAQ-Atendimento  
> Complementa: mapa-temas-gaps-novatech.md e analise-inconsistencias-novatech.md  
> Base: Exercício 1.1.1 — Etapas 1 e 2 + FAQ-atendimento.md

---

## 1. Cruzamento IC × FAQ — inconsistências já catalogadas

### IC-01 × FAQ item 8 — Conflito PROC-042 v1 vs v2
**Classificação do cruzamento:** ⚠️ IC agravada — o FAQ reconhece o conflito mas introduz erro factual novo

| Dimensão | Conteúdo |
|---|---|
| IC-01 identificou | v1 e v2 coexistem sem hierarquia formal. Fatores divergentes: 1.2 vs 1.15 (faixa 1.001–3.000 kg) e 1.5 vs 1.4 (acima de 3.000 kg). |
| FAQ item 8 diz | "Cuidado: existem duas versões da PROC-042. A mais recente tem multiplicadores mais altos. Na dúvida, use a mais recente (v2), mas se o cliente reclamar do valor, pode ser que o contrato dele ainda esteja na tabela antiga." |
| Resultado do cruzamento | O FAQ reconhece o conflito e orienta corretamente usar a v2 — mas afirma que "a mais recente tem multiplicadores mais altos", o que está errado: a v2 tem multiplicadores menores (1.15 e 1.4 vs 1.2 e 1.5). O time internalizou a lógica invertida. |

**Novo risco identificado:** O atendente que seguiu o FAQ esperava que a v2 gerasse valores mais altos para o cliente. Isso é o contrário do que o PROC-042 v2 realmente produz. Essa expectativa invertida pode ter consolidado a prática de "se o cliente reclamar do valor, reverte para a tabela antiga" — perpetuando o uso irregular da v1 de forma velada.

**Ação adicional:** Além de retirar a v1 do corpus, comunicar ao time que a v2 resulta em valores menores (não maiores) para as faixas de peso afetadas. Isso também ajuda a reduzir reclamações de clientes sobre valores inesperados no frete.

---

### IC-02 × FAQ item 45 — Limiar de desconto divergente
**Classificação do cruzamento:** 🔴 IC agravada — conflito direto com impacto financeiro recorrente

| Dimensão | Conteúdo |
|---|---|
| IC-02 identificou | FAQ define limiar em "mais de 10 fretes/mês". PROC-042 v2 define "a partir de 8 fretes/mês" (5% de desconto) e "acima de 15 fretes/mês" (10% de desconto). O FAQ desconhece o segundo limiar. |
| FAQ item 45 diz | "Para clientes com mais de 10 fretes especiais por mês, existe desconto automático na tabela (veja PROC-042). Para outros casos, encaminhe ao Comercial com justificativa." |
| Resultado do cruzamento | Conflito direto e danoso: clientes com 8 ou 9 fretes/mês têm direito ao desconto de 5% pelo normativo, mas o atendente nega esse desconto por seguir o FAQ. O segundo limiar (15 fretes → 10%) é completamente invisível para o time. |

**Novo risco identificado:** Esse erro tem impacto financeiro direto e recorrente. Todo mês, clientes na faixa de 8 a 10 fretes/mês podem estar sendo cobrados a mais por desconto negado indevidamente. Dada a operação de 320 chamados/dia e ~60% de consulta documental, o número de clientes afetados pode ser significativo.

**Ação adicional:** Verificar histórico de chamados para estimar quantos clientes foram afetados por desconto negado indevidamente. Comunicar e corrigir o entendimento do time antes do go-live do assistente — corrigir apenas o FAQ informal não basta se o comportamento equivocado já está consolidado na equipe.

---

### IC-03 × FAQ — Frete reverso por desistência
**Classificação do cruzamento:** 🔴 Ausência no FAQ é um sinal — comportamento de escalada já existe informalmente

| Dimensão | Conteúdo |
|---|---|
| IC-03 identificou | POL-001 v3.1 define frete reverso com "os mesmos multiplicadores do frete original" sem especificar qual versão do PROC-042 usar. Ambiguidade não resolvível pelo corpus atual. |
| FAQ — nenhum item correspondente | Nenhum dos itens selecionados aborda custo do frete reverso por desistência do cliente. O FAQ cobre carga danificada (item 38) mas não devolução por desistência com custo ao cliente. |
| Resultado do cruzamento | A ausência no FAQ sugere que o time atualmente escala esse tipo de dúvida ao Comercial sem tentar calcular o valor — o que é, na prática, a resposta correta dada a ambiguidade documental. |

**Novo insight:** A escalada informal ao Comercial funciona como mecanismo de segurança não documentado. O assistente de IA precisa replicar esse comportamento: ao receber perguntas sobre custo de frete reverso por desistência, deve escalar ao Comercial ao invés de tentar calcular com base na ambiguidade da POL-001.

---

### IC-04 × FAQ itens 3 e 32 — Carga perigosa sem PROC-043
**Classificação do cruzamento:** ⚠️ Comportamento tácito correto, por razão desconhecida pelo time

| Dimensão | Conteúdo |
|---|---|
| IC-04 identificou | PROC-042 v2 delega frete de carga perigosa acima de 500 kg ao PROC-043, que está em revisão pelo Compliance e ausente do corpus. |
| FAQ item 3 diz | "Na prática, a gente orienta o cliente a ligar no ramal 4500 (Gestão de Riscos). Oficialmente não pode pelo processo padrão, mas já tiveram casos em que o pessoal de Riscos autorizou exceção." |
| FAQ item 32 diz | "Precisa de autorização do Compliance e a documentação ANTT tem que estar atualizada. Na prática, demora uns 2 dias para conseguir a autorização." |
| Resultado do cruzamento | O FAQ capturou a orientação prática correta (escalar para Gestão de Riscos / Compliance), mas sem saber o motivo real: o PROC-043 está em revisão e não há regra publicada. O time descobriu empiricamente a resposta certa pelo caminho errado. |

**Novo insight:** O ramal 4500 (Gestão de Riscos) mencionado no FAQ item 3 coincide com o canal indicado na POL-001 v3.1 para carga perigosa fora do processo padrão. Isso é uma consistência valiosa — o assistente pode usar essa orientação de escalada com segurança, mesmo sem o PROC-043 disponível, citando a POL-001 v3.1 como fonte formal.

**Ação adicional:** Configurar o assistente para reconhecer perguntas sobre frete de carga perigosa acima de 500 kg e responder com orientação de escalada (ramal 4500 / Compliance), sem tentar calcular valor ou prazo até o PROC-043 ser publicado e indexado.

---

### IC-05 × FAQ item 38 — Carga danificada: sinistro vs devolução
**Classificação do cruzamento:** ✅ IC parcialmente resolvida — o FAQ confirma que são processos distintos

| Dimensão | Conteúdo |
|---|---|
| IC-05 identificou | FAQ (48h, e-mail Jurídico) e POL-001 v3.1 (7 dias úteis, Portal do Cliente) descrevem fluxos totalmente diferentes. Hipótese: são processos distintos — sinistro vs devolução. |
| FAQ item 38 diz | "Carga danificada em trânsito tem processo diferente de devolução. O cliente precisa registrar a ocorrência em até 48h após o recebimento, com fotos e laudo se possível. (...) isso passa pelo Jurídico, não pelo atendimento normal — encaminhe para sinistros@novatech.com.br." |
| Resultado do cruzamento | O próprio FAQ confirma explicitamente que carga danificada é "processo diferente de devolução". A hipótese da IC-05 está correta: são dois fluxos distintos. A IC-05 não é uma inconsistência entre documentos — é uma lacuna de documentação formal. |

**Resolução parcial:** O processo de sinistros (carga danificada em trânsito) existe e funciona na prática, mas só está documentado no FAQ informal. O assistente precisa distinguir os dois fluxos pela natureza da pergunta do atendente:

- "Carga danificada antes/durante a entrega" → processo de sinistros → sinistros@novatech.com.br (48h para registro)
- "Devolução após entrega" → POL-001 v3.1 → Portal do Cliente (7 dias úteis)

**Ação adicional:** Solicitar à área Jurídica a formalização do processo de sinistros como documento normativo independente da POL-001. Até a publicação, tratar o item 38 do FAQ como referência válida para sinistros — com sinalização de que é fonte informal — e a POL-001 como referência exclusiva para devoluções.

---

### IC-06 × FAQ — Versionamento da POL-001
**Classificação do cruzamento:** 🔵 Ausência no FAQ revela padrão organizacional — o time nunca usou a POL-001

| Dimensão | Conteúdo |
|---|---|
| IC-06 identificou | Versão truncada da POL-001 pode ainda coexistir com a v3.1 no SharePoint, criando ruído no índice. |
| FAQ — nenhum item correspondente | O FAQ não referencia a POL-001 em nenhum dos 9 itens selecionados. Devolução aparece apenas em fragmentos nos itens 3 e 38, sem citar a política oficial. |
| Resultado do cruzamento | A ausência total da POL-001 no FAQ confirma que o time de atendimento nunca usou esse documento como referência. O FAQ substituiu integralmente a política formal no dia a dia — o que explica por que a versão truncada passou despercebida por meses. |

**Novo insight — risco organizacional:** Esse padrão provavelmente se repete em outros temas além de devolução. O assistente de IA vai introduzir pela primeira vez um canal que força o atendente a consultar a documentação oficial. Isso pode gerar resistência se as respostas do assistente diferirem do que o time aprendeu a responder empiricamente — especialmente nos pontos onde o FAQ está errado (IC-02) ou simplificado (IC-01).

---

## 2. Itens do FAQ sem inconsistência catalogada — análise complementar

Itens do FAQ que não conflitam com documentos normativos analisados, mas apresentam riscos próprios para o assistente de IA.

### FAQ item 15 — Tier Platinum inexistente
**Status:** ✅ Alinhado ao SLA-2024

O FAQ orienta corretamente: tiers são Gold, Silver e Standard; Platinum foi descontinuado em 2022. O SLA-2024 confirma que não existem outros tiers além dos três listados. Pode ser indexado com segurança, preferencialmente citando o SLA-2024 como fonte primária.

### FAQ item 22 — Seguro de carga (0,3% padrão / 0,8% perigosa)
**Status:** ⚠️ Sem fonte normativa — risco de confiança indevida

Nenhum documento normativo no corpus valida ou contradiz esses percentuais. O FAQ adiciona o alerta "contratos mais antigos podem ter percentuais diferentes — confirme com o Comercial", o que é a orientação correta de escalada. O assistente não deve apresentar esses valores como fato estabelecido sem sinalizar que a fonte é informal e que contratos anteriores a 2023 podem diferir.

### FAQ item 27 — Tracking parado há 5 dias
**Status:** ⚠️ Sem procedimento normativo — critério de R$ 50k não validado

A orientação prática (abrir chamado de rastreamento, prioridade alta para clientes Gold ou valor de carga acima de R$ 50.000) não tem procedimento normativo correspondente no corpus. O critério de R$ 50.000 para classificação de prioridade não aparece no SLA-2024 nem em qualquer outro documento analisado. Indexar apenas com metadado de "fonte informal" ou configurar escalada para esse tema.

### FAQ item 41 — SLA de resposta vs resolução
**Status:** ✅ Alinhado ao SLA-2024

Definições e valores corretos: Gold 2h resposta / 24h resolução, Silver 4h / 48h, Standard 8h / 72h. Alinhado ao SLA-2024. Pode ser indexado com segurança, citando o SLA-2024 como fonte primária.

---

## 3. Resumo do cruzamento por tipo de resultado

| Tipo de resultado | ICs / Itens | Descrição |
|---|---|---|
| 🔴 IC agravada pelo FAQ | IC-01, IC-02 | O FAQ introduz erro factual que piora o risco original |
| ⚠️ Comportamento tácito correto, razão errada | IC-04 | O time chegou à resposta prática certa sem saber o motivo real |
| ✅ IC parcialmente resolvida | IC-05 | O FAQ confirma a hipótese de processos distintos |
| 🔵 Ausência no FAQ como sinal | IC-03, IC-06 | A ausência revela padrão de escalada informal ou não uso do documento |
| ✅ FAQ alinhado ao normativo | itens 15, 41 | Podem ser indexados com segurança (fonte primária = documento oficial) |
| ⚠️ FAQ sem contrapartida normativa | itens 22, 27 | Sem fonte oficial — indexar com metadado de "fonte informal" ou configurar escalada |

---

## 4. Implicações consolidadas para o design do assistente

O cruzamento entre as inconsistências e o FAQ revela três padrões estruturais que devem orientar o design do assistente:

**Padrão 1 — O FAQ como espelho do comportamento real**
O FAQ não é apenas uma fonte de erros — ele documenta como o time realmente opera. Em vários casos, o time chegou à resposta prática correta (escalar para Gestão de Riscos, encaminhar frete reverso ao Comercial) sem ter a base documental que justifica essa orientação. O assistente deve replicar esses comportamentos de escalada, agora com a citação da fonte normativa correta.

**Padrão 2 — Escalada como mecanismo de segurança não documentado**
Onde o FAQ não tem resposta (IC-03, IC-06) ou onde a resposta é "fale com fulano" (IC-04, IC-05), o assistente deve configurar escalada explícita — não tentar calcular ou inferir. Os pontos de silêncio do FAQ são tão informativos quanto seus erros.

**Padrão 3 — Risco de resistência organizacional no go-live**
O FAQ substituiu a documentação normativa no dia a dia do time por anos. O assistente vai introduzir respostas que, em alguns casos, contradizem o que o time aprendeu a responder (especialmente IC-02, onde o limiar de desconto correto é diferente do que o time pratica). A estratégia de go-live deve incluir comunicação proativa ao time sobre as correções — não apenas a implantação técnica do assistente.

---

*Documento produzido na fase de discovery do projeto NovaTech × DB1*  
*Complementa: mapa-temas-gaps-novatech.md e analise-inconsistencias-novatech.md*  
*Próximo passo: definição da estratégia de curadoria, metadados de indexação e política de escalada do assistente*
