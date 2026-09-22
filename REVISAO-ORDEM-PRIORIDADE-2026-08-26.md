# Revisão — onde mais a régua "plug and play" se aplica

**Data:** 2026-08-26
**Contexto:** junto com a implementação da ordenação por prioridade de conversão na Lista Detalhada dos Pacientes, o pedido foi *"aproveite para revisar o código e checar se isso é passível de aplicar regras nos outros KPIs e páginas"*.

Cada item tem o ponto exato do código, o esforço e o que precisa ser decidido antes.

## Status (revisado em 2026-09-21)

| # | Item | Status |
|---|---|---|
| 1 | Lista de Pacientes Potenciais (Ganho) | ✅ **FEITO em 2026-09-21** — por prioridade de status, escolha explícita do usuário |
| 2 | Pendências de Faturamento | ⏳ aberto — falta confirmar a régua ("mais antiga primeiro") |
| 3 | Avaliações de NPS | ⏳ aberto — falta checar redundância com o card de Destaques |
| 4 | Unificar as 3 réguas de especialidade | ⏳ aberto — muda a ordem dos cartões do Ganho, precisa de decisão |
| 5 | Divergência Insights × Lista | ⏳ aberto — **decisão de negócio**, não técnica |
| 6 | Consolidado das Clínicas | ✅ encerrado sem ação (já tem ordenação clicável; recomendado não mexer) |

⚠️ Conferido item a item contra o código em 2026-09-21: nada dos itens 2 a 5 foi implementado nesse meio-tempo — `faturamentoFilteredRecords` e `npsFilteredData` continuam saindo na ordem bruta, as 3 réguas de especialidade continuam existindo e a divergência do item 5 continua real.

---

## 0. O que JÁ foi implementado (base desta revisão)

`sortByConversionPriority()` ordena a Lista Detalhada dos Pacientes (tela **e** as 3 exportações) em 3 camadas, nesta precedência:

1. **Status do procedimento ativo** — grupos 1 a 8 (`CONVERSION_PRIORITY_RANK` + regra do 2º olho); "aguardando retorno" e "sem status" dividem o mesmo rank 4
2. **Atrasado antes de em andamento** — atrasado → vence hoje → futuro → sem data
3. **Especialidade** como desempate — `SPECIALTY_PRIORITY_ORDER` (primária > secundária > terciária, com ordem definida dentro de cada nível)

Aplicada num lugar só: `getSearchedPatientData()`, que é a fonte única da tela e das exportações.

---

## 1. Lista de Pacientes Potenciais para Operar (Potencial de Ganho) — ✅ FEITO em 2026-09-21

> **Implementado**, com a decisão pendente resolvida pelo usuário: **prioridade de status**, não R$ (a recomendação abaixo). `getSearchedGanhoPatientData()` agora envolve o retorno em `sortByConversionPriority()` — uma linha, no ponto único que alimenta tela e as 3 exportações. O resto desta seção fica como registro do diagnóstico original.

**Situação antes:** saía na ordem bruta da planilha. É exatamente o mesmo tipo de lista de trabalho da Lista Detalhada — o responsável da clínica abre pra atacar convertíveis, mas não tem nenhuma pista de por onde começar.

**Por que é o candidato mais forte:** a estrutura é idêntica à de Consultas — `getSearchedGanhoPatientData()` é a fonte única de tela + xlsx + CSV + PDF. É literalmente **uma linha** (envolver o retorno em `sortByConversionPriority()`), exatamente como foi feito em `getSearchedPatientData()`.

**Como a régua se comporta lá:** `getGanhoConvertiblePatients()` já exclui quem realizou e quem está sem indicação, então dos 8 grupos só sobram os ranks **1, 2, 3, 4 e 7 (desistiu)** — os grupos 5, 6 e 8 nunca aparecem. A régua funciona sem adaptação nenhuma.

**Decisão pendente (é o único bloqueio):** ordenar por **prioridade de status** (mesma régua) ou por **maior R$ potencial primeiro**?
- *Recomendação:* mesma régua de status. O valor em R$ já está evidenciado nos cartões acima; a lista existe pra ação, e ordenar por dinheiro colocaria uma catarata "sem indicação de urgência" à frente de um caso quente prestes a esfriar.
- *Alternativa híbrida:* rank de status manda, e dentro do mesmo rank+atraso, maior valor primeiro (substituindo o desempate por especialidade). Faz sentido se a clínica pensa por faturamento, não por fila.

---

## 2. Pendências de Faturamento — **ALTA prioridade, mas com régua PRÓPRIA**

**Situação hoje:** a tabela sai na ordem em que `getPendingBillingRecords()` varre a planilha (paciente por paciente, aba por aba). Sem priorização.

**Por que a régua de conversão NÃO serve aqui:** todo registro dessa página já é `CIRURGIA_REALIZOU_*` — todos cairiam no mesmo rank 6. Não é conversão, é **dinheiro parado**: a cirurgia já aconteceu, o FEE não foi pago.

**Régua sugerida:** **data da cirurgia crescente** — a mais antiga primeiro. Quanto mais velha a pendência, maior o risco de a clínica já ter fechado o mês sem reportar e o valor virar prejuízo. Desempate por clínica (agrupa o trabalho de conferência de quem vai cobrar).

**Ganho colateral:** hoje, pra saber qual pendência é mais crítica, é preciso baixar a planilha e ordenar na mão. Com isso, o topo da tela já é a fila de cobrança.

**Onde mexer:** `faturamentoFilteredRecords` (montado em `applyFaturamentoFilters()`) — mesma lógica de fonte única: ordenar ali cobre a tela e as 3 exportações.

**Esforço:** baixo (~5 linhas). **Precisa de decisão:** confirmar que "mais antiga primeiro" é a régua certa, e se o desempate por clínica atrapalha quem lê por paciente.

---

## 3. Avaliações Detalhadas de NPS — **MÉDIA prioridade**

**Situação hoje:** `npsFilteredData` sai na ordem da planilha, com promotores e detratores embaralhados.

**Régua sugerida (mesma filosofia):** Detrator → Neutro → Promotor → Nota Ausente; dentro de cada faixa, **menor nota primeiro** e depois avaliação mais recente primeiro (dá pra agir enquanto a experiência ainda é fresca pro paciente).

**Cuidado antes de fazer:** o card **"Destaques que Precisam de Atenção"** já faz um recorte parecido (palavras-chave negativas + categorias marcadas como "não gostei"). Vale checar se ordenar a tabela vira **redundância** ou se são coisas complementares — o card é uma seleção qualitativa, a tabela seria a fila completa. Minha leitura: são complementares, mas é chamada do usuário.

**Esforço:** baixo. **Onde:** `npsApplyFilters()`, no ponto em que `npsFilteredData` é montado.

---

## 4. Dívida técnica — hoje existem **3 réguas de especialidade** no código

Este item não é uma funcionalidade nova, é uma inconsistência que a implementação de hoje deixou visível:

| Régua | Onde | O que define |
|---|---|---|
| `SPECIALTY_RULES[].tier` | Insights | nível (primária/secundária/terciária) + metas |
| `getSpecialtyTierOrder()` | Potencial de Ganho | **só** o nível — dentro do nível, desempata por ganho R$ ou alfabético |
| `SPECIALTY_PRIORITY_ORDER` | Lista Detalhada (novo) | nível **+ a ordem exata dentro do nível** |

**O problema concreto:** a ordem dentro do nível fechada agora (catarata > refrativa > blefaroplastia > retina; capsulotomia > pterígio > calázio > consulta oftalmológica; estrabismo > glaucoma > ceratocone > córnea) **não bate** com a ordem de declaração de `SPECIALTY_RULES` (lá calázio vem antes de pterígio, glaucoma antes de estrabismo). Hoje isso não quebra nada porque o Ganho só usa o `tier`, mas é o tipo de divergência silenciosa que o `classifyStatus()` foi centralizado pra evitar.

**Proposta:** fazer `getSpecialtyTierOrder()` derivar de `SPECIALTY_PRIORITY_ORDER` (uma fonte só), e usar a ordem completa como **desempate final** nos cartões do Ganho.

**Atenção — muda tela:** hoje os cartões do Ganho ordenam por *maior ganho potencial primeiro* dentro do nível. Aplicar a ordem de especialidade como critério principal mudaria essa ordem visual. Precisa de decisão explícita: o cartão do Ganho é um **ranking financeiro** (fica como está) ou uma **lista de prioridade** (passa a seguir a régua)? Minha leitura: é ranking financeiro, então o desempate por ordem de especialidade só deveria entrar quando dois valores empatam.

---

## 5. ⚠️ Divergência REAL entre a ordem dos Insights e a nova ordem da lista

Achado da revisão, e o item que mais merece uma decisão sua — porque hoje o painel **fala duas coisas diferentes sobre o que é mais urgente**:

| # | Ordem das recomendações dos Insights (`buildRecommendations`) | # | Nova ordem da Lista Detalhada |
|---|---|---|---|
| 1 | Cirurgia **agendada** atrasada | 1 | **Aguardando a clínica agendar** |
| 2 | Aguardando a clínica agendar | 2 | Cirurgia agendada atrasada |
| 3 | Exame com retorno pendente atrasado | 3 | Exames atrasados (todos juntos) |
| 4 | **2º olho de catarata pendente** | 4 | Aguardando retorno **e** sem status (empatados) |
| 5 | Exame oftalmológico atrasado | 5 | **2º olho de catarata pendente** |
| 6 | Exame clínico/laboratorial atrasado | 6-8 | já operou → desistiu → sem indicação |
| — | *(não tem recomendação pra "aguardando retorno" nem "sem status")* | — | — |

**As 3 divergências:**
1. **Os 2 primeiros estão trocados.** Insights manda reagendar cirurgia vencida antes de destravar quem aguarda agendamento; a lista faz o contrário.
2. **O 2º olho perdeu posição.** Nos Insights ele foi promovido em 2026-08-11 justamente por ser "o item de maior oportunidade de conversão rápida" — na nova ordem ele vem depois dos exames e depois de "aguardando retorno / sem status".
3. **"Aguardando retorno" e "sem status" só existem na lista.** Os Insights não geram recomendação pra eles, embora "sem status" seja documentado como o **pior caso** (não se sabe onde o paciente está).

**O que fazer:** as duas ordens podem legitimamente ser diferentes — uma é *recomendação de gestão* (o que a clínica deve priorizar como processo) e a outra é *fila de trabalho paciente a paciente*. Mas isso precisa ser **uma decisão consciente**, não um acidente. Se forem para ser a mesma coisa, o certo é extrair uma constante única de prioridade e as duas telas lerem dela.

---

## 6. Relatório Consolidado das Clínicas — **BAIXA / provavelmente não mexer**

Já tem ordenação clicável por qualquer coluna (implementada em 2026-08-12), com default "maior volume total primeiro", e a exportação segue o que está na tela. Um default por "Aguardando Clínica" (a coluna que mais trava conversão) seria mais "plug and play", **mas mudaria silenciosamente a planilha de quem já usa o relatório hoje**. Sugestão: deixar como está — quem quer a visão de prioridade já consegue com um clique no cabeçalho.

---

## Resumo — ordem sugerida de execução

| # | Item | Esforço | Bloqueio |
|---|---|---|---|
| 1 | Lista de Pacientes Potenciais (Ganho) | ~1 linha | escolher status × R$ |
| 2 | Pendências de Faturamento (mais antiga primeiro) | baixo | confirmar régua |
| 5 | Alinhar (ou assumir) a divergência Insights × Lista | médio | **decisão de negócio** |
| 3 | Avaliações de NPS (detratores primeiro) | baixo | checar redundância com Destaques |
| 4 | Unificar as 3 réguas de especialidade | médio | muda ordem dos cartões do Ganho |
| 6 | Consolidado das Clínicas | — | recomendo não mexer |
