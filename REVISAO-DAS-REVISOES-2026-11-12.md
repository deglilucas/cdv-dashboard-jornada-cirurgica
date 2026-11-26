# Parecer sobre as duas revisões — conferido contra o código de 2026-11-12

Documentos conferidos: `REVISAO-CODIGO-2026-08-07.md` e `REVISAO-ORDEM-PRIORIDADE-2026-08-26.md`.
Base da conferência: `index.html` na branch `feature/data-geracao-pdf-listas` (7.827 linhas, 268 funções, 6 abas).

Nada foi alterado no código. Este arquivo só registra o que vale, o que pode sair e o que falta.

## Resumo

| Documento | Situação | O que fazer |
|---|---|---|
| `REVISAO-ORDEM-PRIORIDADE-2026-08-26.md` | Os 6 itens estão fechados e o código confere | Arquivar (ou apagar). Só um texto dele ficou desatualizado (item 3) |
| `REVISAO-CODIGO-2026-08-07.md` — Fases 1 e 2 | Feitas e ainda de pé no código, com 3 sobras pequenas | Manter só as sobras (seção 2) |
| `REVISAO-CODIGO-2026-08-07.md` — Fase 3 | Não foi executada e o plano escrito não serve mais | Decidir de novo se ainda quer; se sim, replanejar do zero (seção 3) |
| O que nenhum dos dois cobre | 9 pontos | Seção 4 |

---

## 1. O que pode ser descartado

### 1.1 `REVISAO-ORDEM-PRIORIDADE-2026-08-26.md` inteiro

Todos os itens conferem com o código:

- Item 1 (lista do Ganho): `getSearchedGanhoPatientData()` ordena por `sortByConversionPriority()`.
- Item 2 (Faturamento): `sortBillingRecords()` existe, pendentes primeiro e mais antiga no topo.
- Item 4 (réguas de especialidade): `SPECIALTY_ORDER_WITHIN_TIER` + `getSpecialtyPriorityOrder()` derivada.
- Item 5 (Insights × Lista): `buildRecommendations()` segue a régua da lista.
- Item 6: sem ação, como combinado.
- Item 3 (NPS): o texto diz "a tabela continua na ordem da planilha — a proposta original não será feita". **Isso deixou de ser verdade em 2026-10-27**: `getNpsSortedData()` ordena a tabela por ponto de atenção (detratores primeiro), que é praticamente a proposta original. O `CLAUDE.md` já registra a mudança; o documento de revisão não.

O `CLAUDE.md` já guarda todas as decisões desse documento, então ele não tem mais função de trabalho. Se for mantido como histórico, corrigir a frase do item 3.

### 1.2 Itens da revisão de código que perderam o objeto

- **Fase 1.A item 2 (eixo 0–100 do gráfico "NPS por Clínica")**: o gráfico não existe mais, foi trocado pelo Ranking de NPS. A barra do ranking já trata nota negativa (`(score + 100) / 2`).
- **Fase 1.A item 1 (NPS negativo como "0,00")**: resolvido e mantido no redesenho. `npsGaugeSvg()` trava só o ponteiro e mostra o número real com sinal.
- **Fase 2 itens 3 a 7 e a observação do `NPS_AVAL_FIELDS`**: todos de pé (`buildPatientRowHtml`, `patientMatchesSearch`, `paginate`/`bindPrevNext`, `*_ALIASES`, `.badge-pill`, `Object.keys(NPS_AVAL_LABELS)`).
- **Mapa de funções e números de linha do documento inteiro**: nenhum número de linha bate mais. Não usar como referência.
- **Checklist de teste do fim do documento**: fala em 4 abas e 10 cards; hoje são 6 abas, 12 cards exportáveis como imagem, 6 listas e a aba de ZIP. Precisa ser reescrito se for continuar em uso.

---

## 2. O que ainda faz sentido (sobras das Fases 1 e 2)

### 2.1 Tailwind sem versão fixa e sem verificação de integridade — média

`index.html:9` ainda carrega `https://cdn.tailwindcss.com` sem versão. O documento marca "Fase 1.B item 3" como feito; os outros 8 scripts estão de fato com versão e hash, o Tailwind não, e o comentário no código assume isso.

É o único script do painel que pode mudar sozinho de um dia para o outro. O próprio Tailwind avisa que esse endereço não é para produção. Duas saídas:

- **Barata**: fixar a versão no endereço (`cdn.tailwindcss.com/3.4.x`). Resolve "muda sozinho", não resolve integridade.
- **Completa**: gerar o CSS uma vez e servir como arquivo estático. Elimina o compilador em runtime e a regra do `!important` que o `CLAUDE.md` documenta, mas cria um passo de geração a cada classe nova. Vai contra o "sem build step" — só vale se a Fase 3 for feita.

Recomendo a barata agora.

### 2.2 Menu "Exportar como…" repetido no HTML — baixa

Era o item adiado da Fase 2 (16 blocos). Hoje são **18 menus** escritos à mão (12 de card + 6 de lista). Continua sendo só repetição de marcação, sem bug. Recomendo manter adiado: gerar por JS mexe em todos os cards e o ganho é só de linhas.

### 2.3 18 funções de exportação que só repassam a chamada — baixa

O dispatcher `exportAs(entity, format)` existe, mas as 18 funções antigas (`exportPatientsAsCSV()` etc.) continuam como atalhos de uma linha, e é por elas que o HTML chama. Funciona; a troca dos `onclick` para `exportAs('patients','csv')` apagaria as 18. Fazer junto com 2.2 ou não fazer.

### 2.4 Escape de HTML e fórmula em planilha — conferido, sem pendência

- Passei por todas as telas criadas depois da revisão (Faturamento novo, Benchmark, ranking de médicos, Ranking de NPS, Destaques Positivos, Baixar relatórios). Todo texto vindo da planilha passa por `escapeHtml()`. Não achei ponto aberto.
- Proteção de fórmula: aplicada só no CSV (`sanitizeForCsvFormula`). No `.xlsx` não é necessária — as duas bibliotecas gravam texto como texto, nunca como fórmula. Os caminhos novos (planilha colorida e ZIP) seguem a mesma regra.

Não testei no navegador com uma planilha maliciosa; a conclusão é de leitura do código.

---

## 3. Fase 3 (dividir o JavaScript em arquivos) — decidir de novo

Estava agendada para 2026-09-23 e não aconteceu. Desde então entraram Benchmark, redesenho do NPS e Baixar relatórios.

| | Quando o plano foi escrito | Hoje |
|---|---|---|
| Linhas do arquivo | 4.169 | 7.827 |
| Linhas de JavaScript | ~2.980 | ~6.240 |
| Funções | ~130 | 268 |
| Abas | 4 | 6 |

O plano de 7 arquivos não tem lugar para Benchmark (~900 linhas) nem Baixar relatórios (~400 linhas), e a premissa "nenhuma leitura cruzada fora de função" precisa ser reconferida: a aba de relatórios mexe nos filtros de todas as outras páginas.

**O que faz sentido**: a divisão ficou mais valiosa (o arquivo quase dobrou) e mais arriscada (mais superfície, mais estado compartilhado). Se for feita, o plano precisa ser refeito do zero, com 9 arquivos em vez de 7, e depois de existir pelo menos o teste mínimo do item 4.1. Sem esse teste, recomendo não fazer.

**O que pode ser descartado**: todo o texto do plano atual (estrutura, listas de função por arquivo, estimativas de linha). Guardar só os 7 "riscos / cuidados na migração", que continuam válidos.

---

## 4. O que está faltando (nenhuma das revisões cobre)

### 4.1 Não existe nenhum teste automático — alta

Toda conferência até hoje foi manual, com planilha real fora do repositório. Com 268 funções e regras que dependem umas das outras (procedimento ativo, conversão por cirurgia, paciente × consulta), cada mudança depende de lembrar o que reconferir.

Sugestão que respeita "nenhuma planilha no repositório": uma página `teste.html` que monta **dados sintéticos em memória** (nomes inventados), roda as funções de cálculo e compara com números esperados — os 4 layouts de planilha, `classifyStatus`, `buildGroupStats`, `getBillingRecords`, `npsComputeStats`, `computeBenchmarkRows`. Abre no navegador, sem instalar nada.

### 4.2 Cabeçalho de planilha depende de maiúscula/minúscula — média

`normalizeHeaders()` (`index.html:4765`) compara o nome da coluna exatamente. O `CLAUDE.md` já registra o efeito: em variantes do export com `Id cliente` / `Paciente (Nome)`, o painel mostra "Sem Nome" e conta paciente por consulta. É a única limitação conhecida que muda número na tela. A correção (comparar sem caixa e sem acento) mexe nos 4 layouts e no NPS — mais um motivo para 4.1 vir antes.

### 4.3 `getItemProp()` desiste cedo na busca sem caixa — baixa

`index.html:2917-2922`: na segunda tentativa, se o primeiro nome candidato casa com uma coluna **vazia**, a função devolve vazio sem tentar os outros candidatos. Só aparece em planilha com caixa diferente e coluna vazia; não vi caso real. Correção de uma linha (exigir valor não vazio também nessa volta).

### 4.4 Decisões marcadas como "minha, ainda não confirmada" no `CLAUDE.md` — precisam de resposta sua

Estão espalhadas e nenhuma revisão as lista:

1. Nome da clínica de referência do Benchmark oculto (`BENCHMARK_NOME_MELHOR_VISIVEL = false`).
2. Colunas "Pacientes" e "Cirurgias" da própria clínica na tabela do Benchmark: ficam ou saem?
3. Ranking consolidado de médicos mostra 5 de cima + 5 de baixo, em vez de "10 primeiros".
4. Card de mediana do Benchmark: leitura do pedido "melhor clínica nas principais subs filtradas".
5. Amarelo `#E3A72F` da faixa intermediária do NPS (o design system não tem âmbar).
6. Rosa `#C8457B` da aba e do botão de NPS.
7. "Categoria marcada como Não" não conta como ponto de atenção no NPS.

### 4.5 `CLAUDE.md` com trechos que se contradizem — média — ✅ corrigido em 2026-11-12

- Linha 110 diz "Não há filtro em cascata (a lista mostra todos os médicos da base)"; a linha 109, logo acima, descreve a cascata por clínica, que é o que o código faz. A 110 é sobra.
- Linha 103 fala em período "nas 4 páginas" e a 270 em "4 barras" de filtro; são 5 com o Benchmark.
- Linha 260: "implementado e testado nas cinco páginas" não cita a aba Baixar relatórios.
- O commit `574dc93` (data e hora de geração no PDF das listas) não está registrado no arquivo.

Como o arquivo é a fonte da verdade lida a cada sessão, contradição ali vira erro de implementação depois.

### 4.6 README atrás do código — baixa — ✅ corrigido em 2026-11-12

Não mencionava a aba **Baixar relatórios** (ZIP por clínica), e a descrição da página de NPS era anterior ao redesenho (dois termômetros, ranking, destaques positivos, tabela ordenada por ponto de atenção). Os dois foram atualizados. A regra do FEE de 25% já estava no README (regra 6) — a primeira versão deste parecer dizia o contrário, por erro meu de busca.

### 4.7 Três "modos de impressão" paralelos — baixa

`setInsightsPrintMode`, `setBenchmarkPrintMode` e `setNpsPrintMode` fazem a mesma coisa (reabrir tudo, capturar, restaurar) e precisam ser lembrados em dois lugares: `beforeprint` e `captureSectionCanvas`. Card novo com lista recolhível que esqueça um dos dois sai cortado na exportação. Um registro único de "cards que expandem ao exportar" tiraria essa armadilha.

### 4.8 Política de privacidade de nomes duplicada — baixa

"Médico de fora aparece só com o primeiro nome / outra clínica não aparece" está implementada duas vezes, no Benchmark (`medicoLabel`) e no Ranking de NPS (`clinicLabel` e vizinhos). Se a regra mudar, é fácil mudar só um lado — justamente a regra que não pode vazar.

### 4.9 Potencial de Ganho ainda lê só a aba 1 — decisão em aberto, não bug

Registrado como escopo deliberado desde 2026-08-12. Fica aqui só porque é a última tela que não segue "procedimento ativo" nem "paciente distinto", e nenhuma revisão cobra uma data para decidir.

---

## 5. Ordem sugerida

| # | Ação | Esforço | Depende de |
|---|---|---|---|
| 1 | Responder as 7 decisões do item 4.4 | só resposta | você |
| 2 | Limpar as contradições do `CLAUDE.md` (4.5) e atualizar o README (4.6) | baixo | — |
| 3 | Fixar a versão do Tailwind (2.1, saída barata) | 1 linha + teste visual | — |
| 4 | Página de teste com dados sintéticos (4.1) | médio | — |
| 5 | Cabeçalho sem depender de caixa (4.2) + `getItemProp` (4.3) | médio | 4 |
| 6 | Registro único de modo de impressão (4.7) e de privacidade de nomes (4.8) | médio | 4 |
| 7 | Decidir sobre a Fase 3; se sim, replanejar (seção 3) | alto | 4 |
| — | Menus e atalhos de exportação (2.2, 2.3) | — | recomendo não fazer |
| — | Arquivar `REVISAO-ORDEM-PRIORIDADE-2026-08-26.md` | — | sua confirmação |
