# Dashboard de Jornada Cirúrgica e NPS

Painel web para acompanhar pacientes na jornada cirúrgica oftalmológica, medir a satisfação (NPS) das clínicas parceiras, controlar pendências de faturamento e estimar o potencial de ganho com pacientes que ainda não operaram.

🔗 **Acesse o painel**: [cdv-cxdashboard.netlify.app](https://cdv-cxdashboard.netlify.app/) (link principal) — ou o [backup no GitHub Pages](https://deglilucas.github.io/cdv-dashboard-jornada-cirurgica/)

## Como funciona

Cada pessoa sobe sua própria planilha (exportada do Farol/Metabase) direto no navegador — nenhum dado é enviado a um servidor ou compartilhado automaticamente entre usuários. É só abrir o link, subir o arquivo e navegar entre as páginas.

Formatos aceitos: `.xlsx`, `.xls` ou `.csv`.

## Privacidade

Todo o processamento acontece no navegador de quem abre a página — a planilha nunca é enviada a um servidor.

O repositório é público (necessário para a hospedagem gratuita), e por isso **não contém nenhum dado de paciente**: planilhas, mesmo de teste, nunca são versionadas. A única coisa guardada localmente é a configuração de valor de cirurgia por clínica, que é preço, não dado de paciente.

A exportação de NPS inclui o telefone (para a clínica poder contatar o paciente), mas deliberadamente **não** inclui e-mail nem endereço — essa base é enviada a clínicas parceiras.

## As 4 páginas

### 1. Consultas
Página principal — é aqui que você sobe a planilha de consultas. Mostra:
- KPIs gerais (quantos pacientes em cada fase da jornada: exames, aguardando clínica, cirurgia agendada, realizada, etc.)
- **Insights** com recomendações de ação priorizadas por especialidade (ex.: "cirurgia agendada com data vencida — reagendar com o paciente"). ⚠️ Não é uma IA de verdade — são regras fixas combinadas com a equipe, sem chamada a nenhum modelo.
- Funil da jornada, gráficos de distribuição por status e por tipo de cirurgia, performance por clínica parceira
- Lista detalhada de cada paciente (telefone, data de nascimento quando disponível, status atual e se está atrasado), **ordenada do caso mais quente para conversão ao mais frio** — a mesma ordem sai na planilha e no PDF

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

## Regras de negócio

Esta seção existe para que a lógica do painel possa ser reimplementada em **outra plataforma** (BI, aplicativo, backend) sem precisar ler o JavaScript. São as regras que mudam o número que aparece na tela — errar qualquer uma produz um painel que parece certo e está errado.

> ℹ️ As réguas numéricas (metas de conversão, faixas de SLA, limiares de atraso) estão na seção [Réguas e parâmetros](#réguas-e-parâmetros), mais abaixo.

### Regras principais

**1. Procedimento ativo — o painel não olha só a primeira cirurgia**
Cada consulta acompanha até **5 abas de procedimento** (o paciente pode operar os dois olhos, ou ter uma segunda cirurgia). Regra de preenchimento: a aba 1 sempre conta, mesmo vazia — vazio é o pior caso; as abas 2 a 5 só contam se tiverem qualquer valor, porque a maioria dos pacientes não terá um segundo procedimento. O **procedimento ativo** é a **última aba preenchida**, e é ela que define status, data e atraso do paciente em todo o painel. Sem essa regra, um paciente que operou o 1º olho e já tem o 2º agendado aparece como "resolvido" e a data da 2ª cirurgia nunca é checada — pode estar vencida há anos sem ninguém ver.

**2. Status da jornada — lista fechada, uma única função de classificação**
Os status vêm de uma lista oficial e fechada de 12 valores. Toda classificação passa por **uma única função**, usada por KPIs, funil, tabelas e ordenação. É a regra mais importante de arquitetura: telas que reclassificam status por conta própria divergem entre si — já aconteceu neste projeto e teve de ser corrigido.
Agrupamentos: exames (5 status distintos), aguardando a clínica agendar, cirurgia agendada, cirurgia realizada (1º e 2º olho), encerrou–desistiu, encerrou–sem indicação, e **"sem status" = status vazio + "aguardando retorno"** — o pior caso, porque não se sabe onde o paciente está.

**3. Atraso ("gargalo") — data marcada que já passou**
Qualquer status ainda em andamento (exame agendado *ou* pendente, aguardando a clínica agendar, cirurgia agendada) **do procedimento ativo** cuja data já passou conta como atraso. Status resolvidos e "sem status" ficam de fora — não há prazo a cobrar. Na tela isso vira 3 estados: data futura (em andamento), vence hoje (necessário atualizar) e vencida (atrasado).

**4. Conversão conta cirurgias, não pacientes** ⚠️ *contraintuitivo*
"Cirurgia realizada" soma **todas** as abas com cirurgia realizada de cada paciente, não só a do procedimento ativo — toda cirurgia feita é uma conversão válida. Duas consequências deliberadas: quem operou os dois olhos conta **2**, e os grupos **deixam de ser mutuamente exclusivos** (o mesmo paciente soma em "realizada" e no gargalo da cirurgia seguinte). Por isso a soma das colunas pode passar do total de pacientes e a taxa de conversão pode passar de 100% — é a régua de benchmark interno, não um bug.

**5. 2º olho de catarata**
Só se aplica a catarata: se a aba 1 é "realizou 1º olho" e nenhuma outra aba tem qualquer valor, o paciente é sinalizado como pendência de acompanhamento. Se a aba 1 já é "realizou 2º olho" (cirurgia isolada), não há follow-up. Se já existe uma aba 2 em andamento, ela vira o procedimento ativo e o alerta genérico não se aplica — o caso passa a ser acompanhado pelo próprio status dela.

**6. Faturamento — trava por aba**
Cada aba de procedimento tem um campo de trava: preenchido = cirurgia já reportada no fechamento financeiro da clínica e a taxa já paga; vazio = pendente. A trava conferida é sempre a **da mesma aba** do procedimento realizado — nunca misturar abas. Por isso 1º e 2º olho aparecem como **duas linhas** de pendência, não uma: cada linha é um procedimento, não um paciente.

**7. Potencial de ganho — quem é "convertível"**
Paciente que ainda não realizou a cirurgia e cujo status não é "sem indicação cirúrgica". **Inclui quem desistiu, de propósito**: desistência é tratada como reversível (pode ser financeira ou momentânea), enquanto "sem indicação" é motivo clínico real. O valor da cirurgia é negociado **por (clínica × especialidade)** — não é um valor único por especialidade —, é digitado manualmente (não existe na planilha) e fica salvo no navegador de quem usa. Combinação sem valor configurado **não entra na soma**, virando um aviso, em vez de ser contada como R$ 0 silenciosamente.

**8. NPS**
Promotor 9–10, neutro 7–8, detrator 0–6. Avaliação com nota ausente (tem avaliação qualitativa, mas não tem nota numérica) fica em um grupo separado e **não entra no cálculo do percentual**. Nota 0 é válida (detrator) — nunca tratar 0 como ausente, nem exportar ausente como 0.

### Réguas e parâmetros

Os valores abaixo são a configuração de referência do painel. Ficam centralizados em constantes, para poderem ser ajustados sem caçar número espalhado pelo código.

⚠️ **Regra de interface:** esses números **nunca aparecem escritos na tela**. O painel é aberto também pelas unidades parceiras, e o que elas veem é o **dado observado** ("Conversão cirúrgica: 25%") mais um **sinalizador qualitativo** ("abaixo do esperado", "em atraso crítico") — nunca o valor da régua. Ao reimplementar, manter essa separação: a régua decide a cor e o texto do alerta, ela não é exibida.

**Meta de conversão cirúrgica** — medida sobre quem **compareceu** à consulta, não sobre quem foi agendado:

| Especialidade | Meta |
|---|---|
| Catarata | 80% |
| Refrativa | 50% |
| Blefaroplastia | 40% |
| Retina | 40% |
| Demais especialidades com meta | 30% |

Especialidades terciárias não têm meta — ficam fora dessa métrica.

**SLA de mediana** — dias entre a consulta e a 1ª cirurgia:

| Faixa | Padrão | Blefaroplastia |
|---|---|---|
| Melhor que o esperado | ≤ 14 dias | ≤ 29 dias |
| Dentro da meta | ≤ 21 dias | ≤ 39 dias |
| Aceitável (sem alarme) | 22–30 dias | 40–50 dias |
| Ponto de atenção | 31–40 dias | 51–65 dias |
| Crítico | ≥ 41 dias | ≥ 66 dias |

Blefaroplastia tem régua mais larga por ser um procedimento eletivo/estético — a decisão do paciente é naturalmente mais lenta.

**Escalonamento de exame atrasado** — dias corridos desde a data marcada que já passou: **5 dias** = ponto de atenção, **15 dias** = ruim, **30+ dias** = crítico.

**Outros limiares**
- "Sem indicação cirúrgica" acima de **30%** dentro de uma especialidade sinaliza possível problema de triagem
- Abaixo de **10 pacientes** no filtro, o painel avisa que a amostra é pequena antes de qualquer conclusão

### Regras secundárias

**Ordem da Lista Detalhada dos Pacientes ("plug and play")**
A lista sai ordenada do lead mais quente para o mais frio, para que o responsável da clínica percorra de cima para baixo resolvendo primeiro o que converte mais rápido. Três camadas, nesta precedência:

1. **Status do procedimento ativo:** aguardando a clínica agendar → cirurgia agendada → exames → *aguardando retorno e sem status (empatados)* → catarata com 2º olho não acionado → demais já operados → desistiu → sem indicação
2. **Dentro do mesmo status:** atrasado → vence hoje → em andamento → sem data
3. **Desempate por especialidade** (níveis abaixo)

A mesma ordem vale na tela e nos arquivos exportados.

**Níveis de especialidade**
- **Primárias:** catarata > refrativa > blefaroplastia > retina
- **Secundárias:** capsulotomia > pterígio > calázio > consulta oftalmológica
- **Terciárias:** estrabismo > glaucoma > ceratocone > córnea > demais

As terciárias ficam **fora das métricas de conversão e SLA**: a chance de indicação cirúrgica é estruturalmente baixa nelas, então "sem indicação" alto ali é esperado, não um alerta.

**Responsável pela ação nas recomendações**
Quando o paciente já tem indicação cirúrgica, quem contata é a **clínica parceira** — é ela que tem a indicação e/ou realiza o exame no próprio local. A **central gestora** entra só no exame clínico/laboratorial (a clínica parceira não faz esse exame) e nas revisões internas de gestão. Tom sempre propositivo, nunca acusatório.

**Presença ≠ jornada**
"Compareceu"/"faltou" é campo de presença na consulta e **não** é status da jornada cirúrgica — nunca misturar os dois. O painel trabalha só com quem compareceu.

### Detalhes de implementação que fazem diferença

- **Fonte única por lista.** Cada lista tem uma função que devolve os dados já filtrados e ordenados; tela e exportações consomem essa mesma função. É o que garante que o arquivo baixado seja idêntico ao que está na tela, inclusive na ordenação — a exportação ignora só a paginação, nunca o filtro.
- **Mais de um layout de planilha.** Existem dois formatos de exportação em uso, com nomes de coluna diferentes. Um mapa de apelidos de cabeçalho normaliza tudo para nomes canônicos **na entrada** — a alternativa (espalhar candidatos de nome por cada função) vira dívida técnica rápido.
- **Datas.** Podem chegar como texto `dd/mm/aaaa` **ou** como número de série do Excel, às vezes com fração de hora. O leitor precisa aceitar os dois.
- **CSV.** Ler com UTF-8 explícito (senão corrompe acentos) e sem reinterpretação automática de tipos (senão datas de texto são lidas no padrão americano mês/dia e corrompem toda data com dia ≤ 12).

## Tecnologia

Arquivo único (`index.html`) — HTML, CSS (Tailwind) e JavaScript puro, sem backend, sem banco de dados, sem etapa de build. Bibliotecas via CDN (Chart.js, SheetJS, jsPDF, Lucide Icons).
