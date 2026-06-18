# Mapa de Temas Cobertos e Hipóteses de Gaps — NovaTech

> Análise documental realizada no contexto do projeto de assistente de IA (DB1 × NovaTech)  
> Base analisada: 5 documentos fornecidos na fase de discovery

---

## 1. Documentos Analisados — Cobertura de Temas

### SLA-2024 — Tabela de SLA por Tipo de Cliente
**Status:** ✅ Contratual | Completo e confiável

| Tema coberto | Status |
|---|---|
| Classificação de tiers (Gold, Silver, Standard) | ✅ Presente |
| Critérios de elegibilidade por tier | ✅ Presente |
| Tempos de primeira resposta e resolução | ✅ Presente |
| Definição de incidente crítico | ✅ Presente |
| Penalidades por descumprimento de SLA | ✅ Presente |
| Medição via Azure DevOps | ✅ Presente |
| Disponibilidade do portal de tracking | ✅ Presente |

---

### PROC-042 v1 — Procedimento de Frete Especial (versão original)
**Status:** ⚠️ Conflito | Vigência indefinida

| Tema coberto | Status |
|---|---|
| Fórmula base de cálculo de frete | ✅ Presente |
| Fatores de peso para cargas acima de 500kg | ✅ Presente (valores: 1.0 / 1.2 / 1.5) |
| Multiplicadores regionais | ⚠️ Referenciados, mas tabela ausente |
| Prazo de entrega para frete especial | ❌ Seção em branco |
| Condições especiais | ❌ Seção em branco |
| Status de vigência formal | ❌ Indefinido — coexiste com v2 sem hierarquia |

---

### PROC-042 v2 — Procedimento de Frete Especial (revisado)
**Status:** ⚠️ Conflito | Revogação da v1 não formalizada

| Tema coberto | Status |
|---|---|
| Multiplicadores regionais atualizados | ✅ Presente (referenciados) |
| Fatores de peso revisados | ✅ Presente (valores: 1.0 / 1.15 / 1.4) |
| Prazo de entrega: +3 dias úteis | ✅ Presente |
| Disposição transitória (chamados a partir de 01/12/2023) | ✅ Presente |
| Condições especiais | ❌ Seção em branco |
| Revogação formal da v1 | ❌ Ausente — ambas coexistem no SharePoint |

---

### POL-001 — Política de Devolução de Mercadorias
**Status:** 🔴 Crítico | Documento truncado

| Tema coberto | Status |
|---|---|
| Objetivo da política | ✅ Presente |
| Escopo de aplicação | ✅ Presente |
| Regras de devolução (prazos, condições, fluxo) | ❌ Ausente — documento aparentemente truncado |
| Exceções listadas (seção 3) | ❌ Ausente |
| Referência a PROC-088 (interceptação de carga) | ⚠️ Mencionado, mas não está no corpus |

---

### FAQ-Atendimento — Perguntas Frequentes do Time de Suporte
**Status:** ℹ️ Informal | Sem validação de Compliance ou Operações

| Tema coberto | Observação |
|---|---|
| Carga perigosa — orientação de atendimento (item 3) | Informal, sem procedimento normativo |
| Frete especial — alerta sobre conflito de versões (item 8) | Reconhece o problema da v1/v2 |
| Tiers — nega existência do Platinum (item 15) | Alinhado ao SLA-2024 |
| Seguro de carga: 0,3% padrão / 0,8% perigosa (item 22) | Sem fonte oficial no corpus |
| Tracking parado há mais de 5 dias (item 27) | Sem procedimento normativo |
| Frete expresso + carga perigosa (item 32) | Sem procedimento normativo |
| Carga danificada — registro em 48h (item 38) | Sem procedimento normativo |
| Diferença entre SLA de resposta e resolução (item 41) | Alinhado ao SLA-2024 |
| Desconto no frete — autonomia do atendente (item 45) | Sem fonte oficial no corpus |

> ⚠️ **Aviso:** Nenhum item do FAQ foi validado contra documentação normativa oficial.

---

## 2. Hipóteses de Gaps — Riscos para o Assistente de IA

### 🔴 Gap Crítico — Conflito entre versões do PROC-042

**Descrição:** PROC-042 v1 e v2 coexistem no SharePoint sem hierarquia formal. Os fatores de peso são diferentes entre as versões (1.2 vs 1.15 para cargas de 1.001kg–3.000kg; 1.5 vs 1.4 para cargas acima de 3.000kg). O assistente pode retornar valores de frete incorretos dependendo de qual documento for recuperado primeiro pelo mecanismo de busca.

**Risco para o RAG:** Resposta factualmente errada — duas fontes contraditórias para a mesma pergunta.

**Hipótese de causa:** Ausência de processo formal de obsolescência documental. A v2 possui disposição transitória mas não revoga explicitamente a v1.

**Ação recomendada:** Definir com a NovaTech qual versão é a vigente e retirar a obsoleta do índice antes do go-live. Considerar metadado de "versão ativa" na estratégia de indexação.

---

### 🔴 Gap Crítico — POL-001 truncada

**Descrição:** A política de devolução de mercadorias está incompleta — contém apenas objetivo e escopo, sem as regras reais de devolução (prazos, condições, fluxo de aprovação, exceções). Devolução é um dos temas mais frequentes nos chamados de atendimento.

**Risco para o RAG:** O assistente não consegue responder perguntas sobre devolução com base neste documento. Risco de alucinação ou resposta em branco.

**Hipótese de causa:** Documento em rascunho ou com erro de exportação. Não há versão alternativa identificada no corpus.

**Ação recomendada:** Solicitar à NovaTech a versão completa da POL-001 antes da indexação. Não indexar o documento truncado.

---

### ⚠️ Gap Alto — Seguro de carga sem fonte normativa

**Descrição:** A única referência a seguro de carga está no FAQ informal (0,3% para cargas padrão e 0,8% para cargas perigosas). Contratos anteriores a 2023 podem ter percentuais diferentes — sem documento oficial para validação.

**Risco para o RAG:** Confiança indevida em informação não validada.

**Hipótese de causa:** Documento de política de seguros não incluído no corpus ou nunca formalizado.

**Ação recomendada:** Solicitar documento normativo de seguros à área Comercial. Enquanto não existir, configurar o assistente para escalar essa pergunta a um atendente.

---

### ⚠️ Gap Alto — Carga perigosa sem procedimento normativo

**Descrição:** Regras para devolução de carga perigosa, frete expresso com carga perigosa e documentação ANTT estão presentes apenas no FAQ informal. Nenhum procedimento normativo foi identificado no corpus.

**Risco para o RAG:** Resposta sem citação de fonte oficial. Alta sensibilidade regulatória e de segurança.

**Hipótese de causa:** PROC-088 (mencionado na POL-001) e possíveis outros PROCs relacionados a cargas perigosas não foram incluídos no corpus.

**Ação recomendada:** Identificar e incluir todos os PROCs relacionados a cargas perigosas. Acionar área de Compliance para validação antes da indexação.

---

### ⚠️ Gap Alto — Tabela de multiplicadores regionais ausente

**Descrição:** Ambas as versões do PROC-042 referenciam uma tabela de multiplicadores regionais (seção 2.1) que não consta em nenhum dos documentos analisados. Sem ela, o cálculo de frete especial é impossível de ser completado.

**Risco para o RAG:** Resposta de cálculo incompleta — fórmula presente, parâmetros ausentes.

**Hipótese de causa:** Tabela de multiplicadores provavelmente está em planilha separada na pasta de rede (atualização mensal), não incluída no corpus inicial.

**Ação recomendada:** Incluir a planilha de multiplicadores regionais no pipeline de indexação. Considerar estratégia de atualização mensal automatizada dado o ciclo de revisão.

---

### ℹ️ Gap Médio — FAQ como fonte contaminante

**Descrição:** O FAQ não possui responsável, controle de versão nem data única de atualização. Se indexado pelo RAG como fonte de igual peso aos documentos normativos, pode contaminar respostas com informações desatualizadas ou imprecisas.

**Risco para o RAG:** Contaminação do índice — respostas mesclando fontes formais e informais sem distinção.

**Hipótese de causa:** Processo informal de produção de conhecimento no time de atendimento ao longo de 2 anos.

**Ação recomendada:** Definir estratégia de indexação diferenciada para o FAQ — ou não indexá-lo, ou indexá-lo com metadado de "fonte informal" para que o assistente possa sinalizar a limitação na resposta.

---

### ℹ️ Gap Médio — Processo de carga danificada não formalizado

**Descrição:** O FAQ orienta encaminhar casos de carga danificada para sinistros@novatech.com.br com registro em 48h e fotos. Não há procedimento normativo que formalize esse fluxo nem defina etapas após o encaminhamento.

**Risco para o RAG:** Resposta parcial — o assistente pode orientar o primeiro passo mas não consegue detalhar o processo completo.

**Hipótese de causa:** Processo de sinistros documentado na área Jurídica, fora do corpus analisado.

**Ação recomendada:** Solicitar à área Jurídica o procedimento formal de sinistros para inclusão no corpus.

---

### 📝 Observação — Tier Platinum descontinuado

**Descrição:** O FAQ menciona um "programa de fidelidade antigo descontinuado em 2022" (tier Platinum). Sem comunicado oficial de descontinuação no corpus.

**Risco para o RAG:** Ruído histórico — risco baixo, mas clientes antigos podem ainda fazer referência ao tier.

**Ação recomendada:** Incluir comunicado de descontinuação no corpus se existir, ou criar um documento de FAQ oficial que formalize a resposta.

---

## 3. Resumo dos Gaps por Prioridade

| Prioridade | Gap | Ação antes do go-live |
|---|---|---|
| 🔴 Crítico | Conflito PROC-042 v1 vs v2 | Definir versão vigente e retirar a obsoleta |
| 🔴 Crítico | POL-001 truncada | Obter versão completa da política de devolução |
| ⚠️ Alto | Seguro de carga sem norma | Obter documento normativo ou configurar escalada |
| ⚠️ Alto | Carga perigosa sem procedimento | Incluir PROCs de carga perigosa no corpus |
| ⚠️ Alto | Multiplicadores regionais ausentes | Incluir planilha da pasta de rede no pipeline |
| ℹ️ Médio | FAQ como fonte contaminante | Definir estratégia de indexação do FAQ |
| ℹ️ Médio | Processo de sinistros informal | Solicitar procedimento à área Jurídica |
| 📝 Observação | Tier Platinum descontinuado | Incluir comunicado de descontinuação se disponível |

---

*Documento produzido na fase de discovery do projeto NovaTech × DB1*  
*Próximo passo: curadoria documental e definição de estratégia de indexação*
