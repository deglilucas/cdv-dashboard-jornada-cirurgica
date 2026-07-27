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
- **Destaques de NPS** (página NPS): lista de ~35 palavras-chave negativas contextualizadas para cirurgia oftalmológica (`NPS_NEGATIVE_KEYWORDS`) — é um ponto de partida para refinar, não uma lista definitiva. Serve de base para os próximos insights de IA da página de Consultas.
- **Pendente / combinado com o usuário**: os textos de recomendação dos cards de insight ("Recomendação: ...") são um rascunho inicial — o usuário quer revisar a redação junto comigo antes de considerar finalizado. Ainda não fizemos essa revisão.

## Deploy

- Repositório: `https://github.com/deglilucas/cdv-dashboard-jornada-cirurgica` (**público** — decisão deliberada, necessária pro GitHub Pages gratuito funcionar; o código não tem nenhum dado de paciente embutido, então isso é seguro).
- **Netlify** (link principal usado pela equipe): `https://cdv-cxdashboard.netlify.app/` — conectado via "Import from Git", publica automaticamente a cada push/merge na `main`.
- **GitHub Pages** (link de backup): `https://deglilucas.github.io/cdv-dashboard-jornada-cirurgica/` — também publica automaticamente a partir da `main`.
- Fluxo de trabalho: criar branch (`feature/...`), commit, push, abrir PR, e só mesclar (merge) na `main` depois de confirmar com o usuário — merge = publicação imediata nos dois links.
- `gh` (GitHub CLI) está instalado e autenticado localmente nesta máquina (login `deglilucas`, device-flow OAuth). Push via git também já está configurado (Windows Credential Manager + Git Credential Manager).

## Preferências do usuário

- **Sempre confirmar antes de dar `git push` ou mesclar PR** — publicar vai direto pro ar pros dois links, o usuário quer aprovar antes.
- Ao fazer mudanças no código, explicar o que mudou e testar no navegador (via Browser tool) antes de reportar como concluído.
