# Comparação de Avaliações — Manual vs IA

> **Contexto:** duas avaliações independentes das mesmas 6 respostas do assistente em staging
> **Avaliação A:** manual, feita pelo Product Specialist (Exercício 3.1 — Avaliação Própria)
> **Avaliação B:** feita pela IA (Claude), rastreada ao Anexo A
> **Objetivo:** comparar os dois julgamentos, identificar convergências e divergências, e determinar qual está correto segundo o Anexo A

---

## Quadro comparativo

| # | Pergunta (resumo) | Avaliação Manual | Avaliação IA | Concordância? |
|---|---|---|---|---|
| 1 | Prazo de devolução standard | Parcialmente correta — fonte incorreta | ⚠️ Parcialmente correta | ✅ Concordam |
| 2 | Prazo de resolução Silver | Correta | ✅ Correta | ✅ Concordam |
| 3 | Devolução carga perigosa classe 3 | Correta | ⚠️ Parcialmente correta | ❌ Divergem |
| 4 | Política de carga danificada | Correta | 🔴 Incorreta | ❌ Divergem |
| 5 | SLA tier Enterprise | Correta | ✅ Correta | ✅ Concordam |
| 6 | Carga perigosa frete expresso | Correta | 🔴 Incorreta | ❌ Divergem |

**Convergência:** 3 de 6 (respostas 1, 2, 5)
**Divergência:** 3 de 6 (respostas 3, 4, 6)

---

## Análise das convergências

### Resposta 1 — concordância total
Ambas classificaram como parcialmente correta pelo mesmo motivo: a fonte citada (seção 3.2) não sustenta o prazo de 7 dias, que está na seção 3.1. A avaliação manual identificou "fonte incorreta" e a IA detalhou que a seção 3.2 trata justamente das exceções. As duas chegaram à mesma conclusão.

### Resposta 2 — concordância total
Ambas classificaram como correta. O prazo de 48h para resolução de chamado geral do cliente Silver corresponde à SLA-2024. Não há divergência.

### Resposta 5 — concordância total
Ambas classificaram como correta. O tier Enterprise não existe, o assistente reconheceu isso, listou os tiers reais e declarou confiança Baixa. Comportamento exemplar reconhecido pelas duas avaliações.

---

## Análise das divergências

As três divergências seguem um padrão claro: em todas, a avaliação manual foi **mais permissiva** (classificou como correta) e a avaliação da IA foi **mais rigorosa** (parcialmente correta ou incorreta). Analisando cada caso contra o Anexo A:

---

### Resposta 3 — Devolução de carga perigosa classe 3

**Manual:** Correta
**IA:** Parcialmente correta

**Quem está correto:** a avaliação da IA, mas a divergência é de grau.

**Análise:** ambas concordam que o conteúdo central está certo — carga perigosa classe 3 não pode ser devolvida pelo processo padrão, conforme POL-001 seção 3.2. A divergência está no canal de escalada. A resposta do assistente recomenda "escalar para o supervisor", enquanto a POL-001 seção 3.2 especifica que essas categorias devem ser encaminhadas ao setor de Gestão de Riscos (ramal 4500).

A avaliação manual considerou o conteúdo correto e não penalizou o canal genérico. A avaliação da IA penalizou porque o documento especifica o canal correto e os guardrails formalizados (C-P07) exigem canal específico, não genérico.

**Veredito:** a classificação da IA (parcialmente correta) é a mais defensável segundo o Anexo A, porque o documento de fato especifica um canal diferente do indicado pela resposta. Mas é uma divergência legítima de severidade — um avaliador pode razoavelmente considerar que acertar o conteúdo sobre carga perigosa é o que mais importa, e que o canal genérico é um detalhe menor. A divergência aqui é sobre quão rígido aplicar o critério de canal de escalada.

---

### Resposta 4 — Política de carga danificada durante transporte

**Manual:** Correta
**IA:** Incorreta

**Quem está correto:** a avaliação da IA, de forma inequívoca.

**Análise:** esta é a divergência mais significativa, porque a resposta tem dois problemas objetivos que a avaliação manual não capturou:

1. **Confiança Alta com fonte "Nenhuma".** A resposta não cita nenhuma fonte mas declara confiança Alta. Isso é uma contradição estrutural — uma afirmação factual detalhada sem fonte não pode ter confiança alta.

2. **O conteúdo não existe no corpus.** O Anexo A afirma explicitamente, na seção de gaps: "não existe documento formal (POL ou PROC) sobre tratamento de carga danificada em trânsito — a informação existe apenas no FAQ informal". A resposta afirmou detalhes ("reembolso integral", "negligência da transportadora", "laudo técnico") que não constam em nenhum documento — nem no FAQ informal, que fala em "responsabilidade nossa" e encaminhamento para sinistros@novatech.com.br.

A resposta é uma alucinação: inventou uma política que não existe, com confiança Alta e sem fonte. Classificá-la como "correta" não se sustenta contra o Anexo A.

**Veredito:** a avaliação da IA está correta. Esta divergência não é de grau — é um erro de avaliação na análise manual, que validou uma alucinação sobre tema sensível. É o caso mais importante da comparação.

---

### Resposta 6 — Carga perigosa com frete expresso

**Manual:** Correta
**IA:** Incorreta

**Quem está correto:** a avaliação da IA, de forma inequívoca.

**Análise:** a resposta afirma "Sim, cargas perigosas podem ser enviadas via frete expresso" com confiança Alta, citando o FAQ-Atendimento item 32. O Anexo A trata desse caso especificamente em duas passagens:

1. O FAQ carrega aviso explícito de que não foi validado contra documentos oficiais e deve ser usado com cautela.

2. A seção de conflitos do Anexo A diz: "O FAQ diz que carga perigosa pode ser enviada com frete expresso 'com autorização', mas não existe documento formal (PROC ou POL) que defina esse processo. A informação pode ser prática informal não documentada."

A resposta transformou uma prática informal não documentada em uma afirmação categórica ("Sim") com confiança Alta, sobre um tema regulatório sensível (carga perigosa). Os guardrails formalizados (G-N05) proíbem exatamente isso: nunca responder sobre carga perigosa com base apenas no FAQ.

**Veredito:** a avaliação da IA está correta. Como na resposta 4, esta não é uma divergência de grau — a análise manual validou uma resposta de alto risco que afirma com confiança algo sem respaldo normativo sobre tema sensível.

---

## Padrão das divergências

As três divergências têm uma direção consistente: a avaliação manual classificou como "correta" três respostas que a IA identificou como problemáticas. E há uma progressão de gravidade:

| Resposta | Tipo de divergência | Gravidade |
|---|---|---|
| 3 | Divergência de grau (canal de escalada) | Baixa — discutível |
| 4 | Erro de avaliação (alucinação não detectada) | Alta — objetiva |
| 6 | Erro de avaliação (confiança indevida em FAQ) | Alta — objetiva |

A divergência na resposta 3 é defensável dos dois lados — é uma questão de quão rígido aplicar o critério de canal de escalada. Mas as divergências nas respostas 4 e 6 são erros objetivos da avaliação manual: ambas as respostas afirmam, com confiança Alta, informações que o Anexo A identifica explicitamente como não documentadas formalmente.

---

## O que essa comparação revela

### Sobre o risco de avaliação humana sob pressão

As respostas 4 e 6 têm uma característica em comum que explica por que passaram na avaliação manual: **elas parecem plausíveis e bem escritas**. "Reembolso integral mediante laudo técnico" soa como uma política real. "Autorização prévia do compliance e documentação ANTT atualizada" soa como um procedimento legítimo. A fluência da resposta mascarou a ausência de respaldo documental.

Isso é precisamente o risco que o conceito de revisão crítica busca mitigar: respostas fluentes e confiantes recebem menos escrutínio, mesmo quando não têm fonte. A avaliação manual confiou na plausibilidade; a avaliação da IA foi rastrear cada afirmação ao documento e encontrou o vazio.

### Sobre o valor da dupla avaliação

A comparação demonstra o valor de ter duas avaliações independentes. Nenhuma das duas é infalível — mas a divergência entre elas sinaliza exatamente os pontos que merecem uma terceira olhada. Se as duas avaliações tivessem concordado em tudo, as alucinações das respostas 4 e 6 teriam passado despercebidas. A discordância é o que trouxe esses casos à tona.

### Sobre o que isso significa para o harness

As respostas 4 e 6 confirmam por que o harness de governança precisa de enforcement determinístico e não pode depender só de avaliação humana:

- A resposta 4 (confiança Alta + fonte Nenhuma) é uma violação estrutural que a camada de validação determinística rejeitaria automaticamente — sem depender de um humano notar.
- A resposta 6 (carga perigosa + FAQ informal) cairia na regra de enforcement que força escalada para temas sensíveis baseados em fonte informal.

Se o harness estivesse aplicado, nenhuma das duas teria chegado ao ponto de precisar de avaliação manual — teriam sido barradas antes. A avaliação humana é uma camada complementar, não a primeira linha de defesa.

---

## Resumo

| Métrica | Valor |
|---|---|
| Respostas avaliadas | 6 |
| Concordância entre as duas avaliações | 3 (respostas 1, 2, 5) |
| Divergência de grau | 1 (resposta 3) |
| Erro objetivo da avaliação manual | 2 (respostas 4 e 6) |
| Direção das divergências | Manual mais permissiva em 100% dos casos |
| Caso de maior risco | Resposta 6 — afirmação categórica sobre carga perigosa sem respaldo normativo |

A avaliação manual e a avaliação da IA concordaram em metade dos casos. Nas divergências, a avaliação da IA se mostrou mais alinhada ao Anexo A — especialmente nas respostas 4 e 6, onde a análise manual validou alucinações sobre temas sensíveis. A lição central não é que a IA avalia melhor que o humano, mas que respostas fluentes sobre temas sem respaldo documental enganam tanto humanos quanto exigem verificação sistemática contra a fonte — que é o que o harness automatiza.

---

*Comparação realizada contra o Anexo A — NovaTech × DB1*
*Avaliação A: manual (Product Specialist) · Avaliação B: IA (Claude)*
