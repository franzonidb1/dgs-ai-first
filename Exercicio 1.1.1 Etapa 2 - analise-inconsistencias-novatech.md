# Análise de Inconsistências Documentais — NovaTech

> Análise cruzada realizada com base nos documentos atualizados: POL-001 v3.1 e PROC-042 v2 completo  
> Complementa o mapa de temas e gaps gerado na fase anterior do discovery

---

## 1. Gaps resolvidos pelos novos documentos

| Gap anterior | Resolução |
|---|---|
| POL-001 truncada | POL-001 v3.1 completa: prazo de 7 dias úteis, fluxo de 5 etapas, custos e exceções por categoria de carga |
| Multiplicadores regionais ausentes | PROC-042 v2 inclui tabela completa: Sul 1.3 / Sudeste 1.1 / Centro-Oeste 1.4 / Nordeste 1.5 / Norte 1.8 |
| Condições especiais do PROC-042 em branco | Regras de desconto por volume (8 e 15 fretes/mês) e aprovação para cargas acima de 5.000 kg agora presentes |
| Carga perigosa — devolução sem norma | POL-001 v3.1 define formalmente que cargas ANTT classes 1–6 não são elegíveis pelo processo padrão (ramal 4500) |

---

## 2. Inconsistências identificadas no cruzamento de documentos

### IC-01 — Fator de peso: PROC-042 v1 vs v2 ainda coexistem
**Severidade:** 🔴 Crítico — bloqueia go-live  
**Documentos em conflito:** PROC-042 v1 (mar/2023) × PROC-042 v2 (nov/2023)

| Aspecto | PROC-042 v1 | PROC-042 v2 |
|---|---|---|
| Fator — 1.001kg a 3.000kg | 1.2 | 1.15 |
| Fator — acima de 3.000kg | 1.5 | 1.4 |
| Prazo de entrega | Seção em branco | Padrão + 3 dias úteis |
| Status formal | Sem revogação | Vigente a partir de 01/12/2023 |

**Risco para o RAG:** Ambos os documentos ainda existem no SharePoint sem hierarquia formal. O mecanismo de busca pode recuperar os dois simultaneamente e gerar cálculo de frete com fatores incorretos ou resposta contraditória na mesma saída. A disposição transitória da v2 trata chamados antigos, mas não revoga a v1 para novos chamados.

**Ação recomendada:** Marcar o PROC-042 v1 como obsoleto no metadado de indexação ou retirá-lo do corpus antes da indexação. Solicitar à Diretoria Comercial formalização da revogação da v1.

---

### IC-02 — Desconto de frete: limiar divergente entre FAQ e PROC-042 v2
**Severidade:** 🔴 Crítico — bloqueia go-live  
**Documentos em conflito:** FAQ-Atendimento (item 45) × PROC-042 v2 (seção 4)

| Aspecto | FAQ item 45 (informal) | PROC-042 v2 (normativo) |
|---|---|---|
| Limiar para desconto automático | Mais de 10 fretes especiais/mês | A partir de 8 fretes/mês |
| Percentual do desconto | Não especificado | 5% sobre o multiplicador regional |
| Segundo limiar | Não mencionado | Acima de 15 fretes/mês → 10% |
| Casos fora do limiar | Encaminhar ao Comercial | Encaminhar ao Comercial com aprovação da Diretoria |

**Risco para o RAG:** Um cliente com 9 fretes/mês tem direito ao desconto de 5% pelo PROC-042 v2, mas o FAQ diz que o limiar é 10. Se o RAG priorizar o FAQ, o atendente vai negar um desconto que é devido. Além disso, o segundo limiar (15 fretes, 10% de desconto) existe apenas no documento normativo — o FAQ simplesmente desconhece essa regra.

**Ação recomendada:** Não indexar o FAQ como fonte para perguntas sobre desconto, ou atribuir peso muito inferior ao de documentos normativos. O PROC-042 v2 deve ser a fonte única para esse tema.

---

### IC-03 — Custo do frete reverso: referência ambígua na POL-001
**Severidade:** 🔴 Crítico — bloqueia go-live  
**Documentos em conflito:** POL-001 v3.1 (seção 3.5) × PROC-042 v2 (seção 2)

**O problema:** A POL-001 v3.1 define que o frete reverso por desistência do cliente é calculado com "os mesmos multiplicadores do frete original". No entanto, não especifica qual versão do PROC-042 usar como referência.

- Para contratos e chamados anteriores a 01/12/2023, o frete original usou os multiplicadores da v1.
- A v2 tem multiplicadores diferentes (menores) e é vigente desde 01/12/2023.
- A expressão "mesmos multiplicadores do frete original" cria ambiguidade: o cálculo do reverso deve seguir a tabela usada no frete de ida, ou a tabela vigente na data da devolução?

**Risco para o RAG:** O assistente não consegue resolver essa ambiguidade — vai ou escolher uma tabela arbitrariamente ou apresentar as duas opções sem conseguir dar uma resposta definitiva. Em ambos os casos, o valor calculado pode ser incorreto.

**Ação recomendada:** Solicitar esclarecimento à Diretoria Comercial/Operações e atualizar a POL-001 com referência explícita: "usar o PROC-042 vigente na data da solicitação de devolução" ou "usar a mesma versão do PROC-042 aplicada ao frete original".

---

### IC-04 — Carga perigosa acima de 500 kg: dependência em PROC-043 instável
**Severidade:** ⚠️ Alto — requer decisão antes do go-live  
**Documentos envolvidos:** PROC-042 v2 (seção 4) → referência ao PROC-043 (ausente do corpus)

**O problema:** O PROC-042 v2 delega o cálculo de frete para cargas perigosas acima de 500 kg ao PROC-043 (Frete de Cargas Perigosas), com a nota explícita de que esse documento "está em processo de revisão pelo Compliance e pode sofrer alterações". O PROC-043 não está no corpus analisado e suas regras atuais são desconhecidas.

**Risco para o RAG:** Perguntas sobre frete de carga perigosa acima de 500 kg não têm resposta no corpus. O assistente pode alucinar valores ou redirecionar para regras desatualizadas. Como o Compliance está ativamente revisando as regras, qualquer informação indexada pode se tornar incorreta antes do go-live.

**Ação recomendada:** Não cobrir esse tema no assistente até a publicação do PROC-043 revisado. Configurar resposta padrão de escalada para o Compliance quando o tema for detectado. Monitorar publicação do PROC-043 e incluir no pipeline de atualização do índice assim que disponível.

---

### IC-05 — Carga danificada: FAQ e POL-001 descrevem fluxos possivelmente diferentes
**Severidade:** ⚠️ Alto — requer clarificação conceitual  
**Documentos em conflito:** FAQ-Atendimento (item 38) × POL-001 v3.1 (seção 3.3)

| Aspecto | FAQ item 38 (informal) | POL-001 v3.1 (normativo) |
|---|---|---|
| Prazo para registro | 48 horas após recebimento | 7 dias úteis |
| Canal de abertura | E-mail sinistros@novatech.com.br | Portal do Cliente |
| Área responsável | Jurídico | Time de atendimento |
| Triagem | Não mencionada | 4 horas úteis |

**Risco para o RAG:** Os fluxos são completamente diferentes — prazo, canal e área responsável divergem em todos os pontos. O cliente pode ser orientado para o canal errado ou criar urgência desnecessária com o prazo de 48h do FAQ.

**Hipótese relevante:** O FAQ e a POL-001 podem estar descrevendo processos distintos sem perceber:
- O FAQ (item 38) pode estar descrevendo **sinistro** — carga avariada em trânsito por responsabilidade da NovaTech, que segue para o Jurídico.
- A POL-001 pode estar descrevendo **devolução** — decisão pós-entrega que passa pelo atendimento.

Se essa hipótese for confirmada, não é uma inconsistência — são dois fluxos legítimos que precisam ser documentados e nomeados separadamente para que o assistente consiga distingui-los.

**Ação recomendada:** Validar com Operações e Jurídico se carga danificada é tratada pela POL-001 (devolução) ou por processo separado de sinistros. Documentar ambos os fluxos antes da indexação com nomenclatura clara e não sobreponível.

---

### IC-06 — Versionamento da POL-001: versão truncada pode ainda existir no SharePoint
**Severidade:** ℹ️ Observação — verificar antes da indexação  
**Documentos envolvidos:** POL-001 sem versão (truncada) × POL-001 v3.1 (15/01/2024)

**O problema:** A versão truncada da POL-001 identificada anteriormente (sem número de versão, contendo apenas objetivo e escopo) pode ainda estar acessível no SharePoint se não foi formalmente arquivada ou excluída. A v3.1 existe e está completa, mas a coexistência das duas versões no repositório criaria ruído no índice.

**Risco para o RAG:** Se a versão truncada for indexada junto com a v3.1, consultas sobre devolução podem recuperar o documento vazio como resultado relevante, degradando a qualidade das respostas sem que o problema seja imediatamente óbvio.

**Ação recomendada:** Confirmar com a equipe responsável se a versão truncada foi arquivada ou excluída do SharePoint. Garantir que apenas a POL-001 v3.1 seja incluída no corpus de indexação.

---

## 3. Resumo executivo — status por inconsistência

| ID | Descrição resumida | Severidade | Bloqueia go-live? |
|---|---|---|---|
| IC-01 | Fatores de peso divergentes entre PROC-042 v1 e v2 | 🔴 Crítico | Sim |
| IC-02 | Limiar de desconto: 10 fretes (FAQ) vs 8 fretes (PROC-042 v2) | 🔴 Crítico | Sim |
| IC-03 | Referência ambígua ao "mesmo multiplicador" no frete reverso | 🔴 Crítico | Sim |
| IC-04 | PROC-043 em revisão — carga perigosa sem cobertura segura | ⚠️ Alto | Requer decisão |
| IC-05 | Carga danificada: fluxos potencialmente distintos sem documentação separada | ⚠️ Alto | Requer clarificação |
| IC-06 | Versão truncada da POL-001 pode ainda existir no SharePoint | ℹ️ Observação | Não, mas verificar |

---

## 4. Implicação para a arquitetura RAG

Os problemas identificados revelam um padrão recorrente: os documentos novos resolveram os **gaps de conteúdo faltante**, mas criaram **inconsistências de fronteira** — pontos onde dois documentos se tocam e discordam sem que nenhum dos dois perceba que está contradizendo o outro.

Isso tem implicação direta na estratégia de indexação:

**Não é suficiente indexar todos os documentos disponíveis.** É necessário:

1. Definir uma política de fonte única por tema — qual documento é a fonte de verdade para cada tipo de pergunta.
2. Atribuir metadados de vigência e hierarquia a cada documento antes da indexação.
3. Tratar o FAQ como conhecimento tácito auxiliar, nunca como fonte normativa.
4. Criar lista de temas com cobertura bloqueada (ex: frete de carga perigosa acima de 500 kg enquanto o PROC-043 estiver em revisão).
5. Implementar estratégia de atualização do índice alinhada ao ciclo mensal de revisão documental da NovaTech.

---

*Documento produzido na fase de discovery do projeto NovaTech × DB1*  
*Complementa: mapa-temas-gaps-novatech.md*  
*Próximo passo: definição da estratégia de curadoria e metadados de indexação*
