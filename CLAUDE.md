# Dashboard de Jornada Cirúrgica Oftalmológica (Central da Visão)

Dashboard estático (`index.html`) para acompanhamento de pacientes em jornada cirúrgica oftalmológica, NPS de clínicas parceiras e pendências de faturamento. Usado pela equipe da Central da Visão.

## Arquitetura

- **Arquivo único**: `index.html` — HTML + CSS (Tailwind via CDN) + JS puro, sem build step, sem backend, sem banco de dados.
- Bibliotecas via CDN: Tailwind, Chart.js, SheetJS (xlsx), Lucide Icons, Google Fonts.
- Todo o processamento acontece no navegador de quem abre a página. Cada pessoa sobe sua própria planilha (`.xlsx`, `.xls` ou `.csv`); nada é compartilhado entre usuários automaticamente.
- **3 páginas/abas** (`switchTab()`): Consultas, NPS, Pendências de Faturamento. Consultas e Faturamento compartilham a mesma base (`rawData`, carregada uma vez no upload principal). NPS tem upload e base própria (`npsRawData`).

## Fontes de dados e formatos aceitos

Os dados vêm de exportações do Metabase (a empresa tem um sistema próprio chamado **Farol**). Já existem **dois layouts de planilha conhecidos** para a página de Consultas, e o código reconhece os dois automaticamente (função `normalizeHeaders()` + mapa `HEADER_ALIASES`):
1. Formato "completo": colunas como `agendamento_data`, `agendamento_cirg_status_clinica`, `clinicas`, `especialidades` etc.
2. Formato "resumido" (export `busca-*.csv` do Metabase): colunas como `Id paciente`, `Data da consulta`, `Cirg data 1 status`, `Paciente (nome)` etc. Tem menos detalhe — **não tem campo de trava (lock)**, então a página de Faturamento não funciona bem com esse formato (tudo aparece como pendente).

Se aparecer um **terceiro formato** de planilha no futuro, o padrão é: adicionar as novas colunas em `HEADER_ALIASES` mapeando pros nomes canônicos já usados no dashboard, em vez de espalhar candidatos novos em cada função.

⚠️ **ARMADILHA IMPORTANTE — `HEADER_ALIASES` é compartilhado pelos DOIS uploads** (consultas e NPS). A planilha de NPS tem colunas `Clínica`, `Médico` e `Especialidade`, que o alias renomeia para `clinicas`, `medicos` e `especialidades`. Ou seja: **dentro do código do NPS, os dados NÃO estão nas chaves originais da planilha**. Ler `getItemProp(item, ['Clínica'])` devolve vazio. Por isso existem os leitores `npsGetClinica()` / `npsGetMedico()` / `npsGetEspecialidade()` / `npsGetPaciente()` / `npsGetTelefone()` — **usar sempre esses**, nunca o nome cru da coluna. Esse foi exatamente o bug de "médico e clínica em branco" na tabela de Avaliações Detalhadas.

**Layout da planilha de NPS** (`TESTE RELATORIO NPS.xlsx`, 19 colunas): `Cliente`, `Telefone`, `Email`, `CEP`, `Endereço`, `Número`, `Bairro`, `Cidade`, `Estado`, `Especialidade`, `Clínica`, `Médico`, `Nota`, `Aval. de Atendimento CDV`, `Aval. de Acompanhamento CDV`, `Aval. Clínica`, `Aval. Médico`, `Data da Cirurgia`, `Data da Aval.`. Detalhes que importam:
- As **datas vêm como número de série do Excel** (ex.: `46162`, e a data de avaliação com fração de hora: `46176.6099`), não como texto `dd/mm/aaaa`. Quem trata isso é o `parseFlexibleDate()` — não presumir string de data.
- A **`Nota` vem numérica** (não string).
- Os campos `Aval. *` só contêm **"Gostei" ou "Não"** — não são texto livre. **Essa planilha não tem coluna de comentário do paciente.** O `npsGetComentario()` aceita os nomes usuais (`Comentário`, `Observação`, `Feedback`, etc.) para quando algum export do Farol trouxer.

**Bugs de CSV já corrigidos** (não reintroduzir): `XLSX.read` precisa de `codepage: 65001` (senão corrompe acentos) e `raw: true` (senão o SheetJS reinterpreta datas de texto no padrão americano mês/dia, corrompendo qualquer data com dia ≤ 12).

## Regras de negócio importantes (não óbvias pelo código)

Fonte: conversas com o usuário + PDF "Descrição de status pós consulta" (lista oficial e fechada de 12 status do Farol).

- **`classifyStatus()`** é a fonte única de verdade para classificar status — usada em KPIs, funil, tabelas, insights. Nunca reintroduzir lógica de `.includes()` duplicada e divergente entre telas (já corrigimos uma inconsistência assim uma vez).
- **"Sem Status" agrupa** status vazio + "AGUARDANDO RETORNO" — é o pior caso, porque não se sabe o status atual do paciente.
- **2º olho de Catarata**: se a aba 1 é "CIRURGIA - REALIZOU 1º OLHO" e nenhuma outra aba (2 a 5) mostra um "REALIZOU", é sinalizado como pendência de acompanhamento (só se aplica a Catarata).
- **Exames atrasados**: só faz sentido checar atraso para status "AGENDADOS" (data real marcada). Status "PENDENTE" usa a data só como controle interno, não indica atraso.
- **Encerrou** é sempre contado separado: "Desistiu" vs "Sem Indicação Cirúrgica" (não são a mesma coisa — desistência é diferente de não ter indicação). Se "Sem Indicação" > 20% dentro de uma especialidade, o dashboard sinaliza como possível problema de triagem.
- **Faturamento** (`agendamento_cirg_data_clinica_lock*`): preenchido = cirurgia já reportada no financeiro mensal da clínica e o FEE já foi pago. Vazio = pendente. Sempre checar o lock **da mesma aba** do procedimento "realizou" correspondente (não misturar abas).
- **Filtro "Status da Jornada"** nunca deve incluir "COMPARECEU"/"FALTOU" — isso é campo de presença da consulta (`agendamento_status`), não status da jornada cirúrgica. Já corrigimos um bug assim uma vez (populateFilterDropdowns tinha um candidato errado).
- **NPS**: Promotor = nota 9-10, Neutro = 7-8, Detrator = 0-6 (confirmado como a régua certa da empresa). "Nota Ausente" (tem avaliação qualitativa mas não tem nota numérica) fica em bucket separado, não entra no cálculo de %.
- **Insights de IA** (página Consultas): **não é uma IA de verdade** — é lógica de regras fixas em JS, sem chamada a nenhum modelo/API. Por isso o título foi renomeado para "Insights - Teste/Em Produção". Focados em 3 especialidades: Catarata, Refrativa, Blefaroplastia. Mostra aviso de "amostra pequena" quando o filtro tem menos de 10 pacientes.
- **Destaques de NPS** (página NPS): lista de ~35 palavras-chave negativas contextualizadas para cirurgia oftalmológica (`NPS_NEGATIVE_KEYWORDS`) — é um ponto de partida para refinar, não uma lista definitiva. Serve de base para os próximos insights de IA da página de Consultas. Cada destaque mostra **telefone de contato** (para acionar o paciente) e, quando existe, o **comentário do paciente** em bloco próprio; sem comentário, mostra **quais categorias** o paciente marcou como "não gostei" (`npsGetCategoriasNegativas()`). `NPS_NON_TEXT_FIELDS` exclui da varredura de palavras-chave os campos de identificação/endereço (nome, e-mail, rua, bairro, cidade, clínica, médico) — eles nunca contêm opinião do paciente e gerariam destaque falso.
- **Upload contextual**: cada página mostra no topo o botão da SUA planilha (`uploadConsultas` / `uploadNps`, alternados em `updateTabNav()`). Antes havia um único botão que continuava visível na página de NPS e dava a entender que servia para carregar o NPS ali. Faturamento usa a mesma base de Consultas, então compartilha aquele botão. **Não voltar a ter um upload só, nem duplicar o input** — os ids `fileInput` e `npsFileInput` precisam ser únicos por causa do binding em `setupEventListeners()`.
- **Dados pessoais na exportação de NPS**: a planilha traz telefone, e-mail e endereço completo. A exportação inclui **telefone** (a pedido do usuário, para a clínica poder contatar o paciente) mas **não** e-mail nem endereço — decisão explícita, já que essa base é enviada para clínicas parceiras.
- **Pendente / combinado com o usuário**: os textos de recomendação dos cards de insight ("Recomendação: ...") são um rascunho inicial — o usuário quer revisar a redação junto comigo antes de considerar finalizado. Ainda não fizemos essa revisão.

## Exportação de relatórios

O modelo antigo era um único botão no header (`exportReport()` + `window.print()`) que jogava a página inteira num "PDF gigante" difícil de trabalhar. Foi substituído por **exportação granular por seção**, com o formato certo para cada tipo de conteúdo:

- **Gráficos e insights → PDF ou JPEG** (imagem): `exportSectionAsPdf()` / `exportSectionAsJpeg()`, ambos em cima de `captureSectionCanvas()` (html2canvas). **9 cards**: Consultas (Insights, Funil, Distribuição por Status, Distribuição por Tipo de Cirurgia, Performance por Clínica) e NPS (NPS Geral/gauge, NPS por Clínica, Avaliação por Categoria, Destaques que Precisam de Atenção).
- **Listas e relatórios → Planilha, CSV ou PDF** (tabela real, nunca imagem): `exportRowsAsSpreadsheet()` / `exportRowsAsCSV()` / `exportRowsAsPdf()` (SheetJS + jsPDF-AutoTable). **4 seções**: Relatório Consolidado das Clínicas, Lista Detalhada dos Pacientes, Avaliações Detalhadas de NPS e Pendências de Faturamento.
- As linhas exportadas vêm de `buildPatientExportRows()` / `buildClinicExportRows()` / `buildNpsExportRows()` / `buildFaturamentoExportRows()`, que reusam `getSearchedPatientData()`, `buildClinicSummary()`, `npsFilteredData`+`npsGetNota()` e `faturamentoFilteredRecords` — as **mesmas** fontes que alimentam a tela. Exportação sempre considera todo o filtro atual, ignorando a paginação. Não duplicar essa lógica de dados.
- A exportação de NPS acrescenta colunas que não existem na tabela em tela: **"Classificação"** (Promotor/Neutro/Detrator/Nota Ausente), para a planilha ser legível sem reaplicar a régua de nota; **"Não Gostou De"**, com as categorias marcadas negativamente; e **"Comentário"**, quando a planilha tiver esse campo. Registro sem nota sai com a coluna Nota **em branco**, nunca 0 — 0 é uma nota válida (detrator).
- O botão "Baixar (.xlsx)" da barra de filtros de Faturamento foi **substituído** por esse dropdown (`downloadFaturamentoReport()` não existe mais).
- Bibliotecas novas via CDN: `html2canvas`, `jspdf`, `jspdf-autotable`.

**Bugs já corrigidos aqui (não reintroduzir):**
1. **Texto "comido" na imagem/PDF** — o html2canvas clona a página num iframe e mede ali as métricas da fonte para posicionar a linha de base do texto. Sem esperar `document.fonts.ready`, ele mede uma fonte de fallback (a Plus Jakarta Sans ainda não carregou), erra a linha de base e desenha o texto mais baixo; onde havia `overflow: hidden` — principalmente o utilitário `truncate` do Tailwind na legenda do donut — as letras apareciam cortadas. Correção: `await document.fonts.ready` **e** o `onclone` que libera o overflow. O `onclone` só mexe em caixas que contêm texto e **nenhum elemento filho**, para não quebrar barras de progresso e cards que usam `overflow-hidden` de propósito.
2. **PDF de seção alta** — quando a janela está estreita os cards empilham e a seção fica mais alta que a página A4. Encolher para caber em 1 página deixava o texto ilegível (o mesmo problema que a exportação por seção veio resolver), então `exportSectionAsPdf()` **recorta a captura em fatias do tamanho da página** e gera múltiplas páginas.
3. **Caixa com rolagem cortada na imagem** — a lista de Destaques do NPS tem `max-h-[420px] overflow-y-auto`, e a imagem exportada não tem rolagem: a captura mostrava só os ~7 itens visíveis e perdia o resto em silêncio. Container assim precisa da classe **`export-expand`**, que o `onclone` usa para soltar a altura na exportação. Se um card novo com rolagem própria for criado, adicionar essa classe.
4. **Arquivo de 0 byte ao exportar aba oculta** — `html2canvas` num elemento dentro de `display:none` devolve canvas vazio e o navegador baixava um arquivo vazio sem avisar. `captureSectionCanvas()` agora checa `getBoundingClientRect()` e lança erro explicando que a página precisa estar aberta.
5. **Planilha em branco** — `hasRowsToExport()` bloqueia a exportação quando o filtro atual não tem nenhum registro, em vez de gerar arquivo vazio.

**Status**: implementado e testado nas **três páginas** (Consultas, NPS e Faturamento). O botão antigo "Exportar / Imprimir Relatório de ..." do header (que gera o PDF único da página inteira) **continua existindo** — decidir com o usuário se remove.

**Cuidado ao testar gráficos no preview/headless**: o Chart.js pinta o primeiro frame dentro de `requestAnimationFrame`, que não roda quando o painel do navegador está oculto/sem compositar. Nesse estado o canvas fica em branco e a imagem exportada sai sem o gráfico — é artefato do ambiente de teste, não bug do código. Para verificar, forçar `chartXInstance.update('none'); chartXInstance.draw();` antes de capturar.

## Deploy

- Repositório: `https://github.com/deglilucas/cdv-dashboard-jornada-cirurgica` (**público** — decisão deliberada, necessária pro GitHub Pages gratuito funcionar; o código não tem nenhum dado de paciente embutido, então isso é seguro).
- **Netlify** (link principal usado pela equipe): `https://cdv-cxdashboard.netlify.app/` — conectado via "Import from Git", publica automaticamente a cada push/merge na `main`.
- **GitHub Pages** (link de backup): `https://deglilucas.github.io/cdv-dashboard-jornada-cirurgica/` — também publica automaticamente a partir da `main`.
- Fluxo de trabalho: criar branch (`feature/...`), commit, push, abrir PR, e só mesclar (merge) na `main` depois de confirmar com o usuário — merge = publicação imediata nos dois links.
- `gh` (GitHub CLI) está instalado e autenticado localmente nesta máquina (login `deglilucas`, device-flow OAuth). Push via git também já está configurado (Windows Credential Manager + Git Credential Manager).

## Preferências do usuário

- **Sempre confirmar antes de dar `git push` ou mesclar PR** — publicar vai direto pro ar pros dois links, o usuário quer aprovar antes.
- Ao fazer mudanças no código, explicar o que mudou e testar no navegador (via Browser tool) antes de reportar como concluído.
