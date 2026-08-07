# Central da Visão — Dashboard de Jornada Cirúrgica e NPS

Painel web para acompanhar pacientes na jornada cirúrgica oftalmológica, medir a satisfação (NPS) das clínicas parceiras, controlar pendências de faturamento e estimar o potencial de ganho com pacientes que ainda não operaram.

🔗 **Acesse o painel**: [cdv-cxdashboard.netlify.app](https://cdv-cxdashboard.netlify.app/) (link principal) — ou o [backup no GitHub Pages](https://deglilucas.github.io/cdv-dashboard-jornada-cirurgica/)

## Como funciona

Cada pessoa sobe sua própria planilha (exportada do Farol/Metabase) direto no navegador — nenhum dado é enviado a um servidor ou compartilhado automaticamente entre usuários. É só abrir o link, subir o arquivo e navegar entre as páginas.

Formatos aceitos: `.xlsx`, `.xls` ou `.csv`.

## As 4 páginas

### 1. Consultas
Página principal — é aqui que você sobe a planilha de consultas. Mostra:
- KPIs gerais (quantos pacientes em cada fase da jornada: exames, aguardando clínica, cirurgia agendada, realizada, etc.)
- **Insights** com recomendações de ação priorizadas por especialidade (ex.: "cirurgia agendada com data vencida — reagendar com o paciente"). ⚠️ Não é uma IA de verdade — são regras fixas combinadas com a equipe, sem chamada a nenhum modelo.
- Funil da jornada, gráficos de distribuição por status e por tipo de cirurgia, performance por clínica parceira
- Lista detalhada de cada paciente (telefone, data de nascimento quando disponível, status atual e se está atrasado)

### 2. NPS
Tem upload próprio (planilha separada de avaliações). Mostra a nota de satisfação geral e por clínica, o detalhamento por categoria avaliada (atendimento, acompanhamento, clínica, médico) e uma lista de destaques que precisam de atenção — comentários ou avaliações negativas, com telefone do paciente para contato.

### 3. Faturamento
Usa a mesma planilha de Consultas (não precisa subir de novo). Lista as cirurgias já realizadas cuja cobrança ainda não foi reportada/paga pela clínica — para a conferência financeira mensal.

### 4. Potencial de Ganho
Também usa a planilha de Consultas. Estima quanto ainda dá para faturar com os pacientes que não operaram e não estão "sem indicação cirúrgica". Você configura o valor da cirurgia por clínica e especialidade, e o painel multiplica pelo número de pacientes convertíveis — com uma lista de quem acompanhar.

## Requisitos mínimos da planilha

Para as páginas de **Consultas, Faturamento e Potencial de Ganho** funcionarem corretamente, a exportação do Farol precisa ter:
- Dados de identificação do paciente e o **status da jornada** preenchido (conforme a lista oficial de status do Farol)
- Os **campos de data** correspondentes a cada status — não basta o status em si
- As colunas de **todas as abas de procedimento** (até 5, usadas por exemplo para acompanhar o 2º olho de catarata e o faturamento) — mesmo que a maioria fique vazia, as colunas precisam existir na exportação

⚠️ O export "resumido" do Metabase (`busca-*.csv`) tem menos detalhe e não traz o campo de trava do faturamento nem as abas extras de procedimento — a página de Faturamento não funciona bem com esse formato (tudo aparece como pendente).

## Exportação

Todo card de gráfico/indicador pode ser exportado em PDF ou JPEG; toda lista detalhada, em Planilha, CSV ou PDF. O botão "Imprimir Painel" no topo gera um PDF só dos gráficos e indicadores (sem as listas longas).

## Tecnologia

Arquivo único (`index.html`) — HTML, CSS (Tailwind) e JavaScript puro, sem backend, sem banco de dados, sem etapa de build. Bibliotecas via CDN (Chart.js, SheetJS, jsPDF, Lucide Icons). Publicado automaticamente no Netlify e no GitHub Pages a cada atualização.
