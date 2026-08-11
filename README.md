# End-to-End HR Data Pipeline & People Analytics Dashboard

Projeto de Engenharia de Dados e People Analytics voltado para análise de turnover e desligamentos.

A solução reúne todo o fluxo, desde a preparação dos dados até a visualização dos indicadores no Power BI. O processo utiliza Apache Hop para ETL, PostgreSQL para armazenamento, Docker para o ambiente, Jenkins para automação da execução e Power BI para modelagem e análise dos dados.

## Arquitetura

```text
                         ┌───────────────┐
                         │  Fonte Dados  │
                         └───────┬───────┘
                                 │
                                 ▼
                         ┌───────────────┐
                         │  Apache Hop   │
                         │      ETL      │
                         └───────┬───────┘
                                 │
                                 ▼
                         ┌───────────────┐
                         │  PostgreSQL   │
                         │    Docker     │
                         └───────┬───────┘
                                 │
                                 ▼
                         ┌───────────────┐
                         │    Power BI   │
                         │   Dashboard   │
                         └───────────────┘

                         ┌───────────────┐
                         │    Jenkins    │
                         │ Orquestração  │
                         └───────┬───────┘
                                 │
                                 └── Execução automática
                                     dos workflows
```

## Tecnologias utilizadas

| Tecnologia | Utilização |
|---|---|
| **Apache Hop** | ETL, pipelines e workflows |
| **PostgreSQL 15+** | Armazenamento dos dados tratados |
| **Docker / Docker Compose** | Containerização do ambiente |
| **Jenkins** | Automação e execução do pipeline |
| **Power BI Desktop** | Modelagem, análise e visualização |
| **DAX** | Medidas e cálculos analíticos |
| **Linux / WSL2** | Ambiente de desenvolvimento |

---

## Estrutura dos dados

Após o processamento, os dados são armazenados na tabela `fato_desligamentos` no PostgreSQL.

A tabela foi estruturada para facilitar o consumo pelo Power BI e permitir análises de volume de desligamentos, tempo de permanência, motivos e características dos colaboradores.

| Coluna | Tipo | Descrição |
|---|---|---|
| `data_desligamento` | TIMESTAMP | Data do desligamento |
| `data_admissao` | TIMESTAMP | Data de admissão |
| `segmento` | VARCHAR | Área ou setor de atuação |
| `motivo` | VARCHAR | Motivo do desligamento |
| `dias_empresa` | NUMERIC | Quantidade de dias na empresa |
| `meses_empresa` | NUMERIC | Tempo de permanência em meses |
| `anos_empresa` | NUMERIC | Tempo de permanência em anos |
| `ano_desligamento` | NUMERIC | Ano do desligamento |
| `mes_desligamento` | NUMERIC | Mês do desligamento |
| `classificacao_tempo_casa` | VARCHAR | Faixa de tempo de permanência |
| `tipo_desligamento` | VARCHAR | Voluntário ou involuntário |

### Classificação do tempo de casa

O tempo de permanência é categorizado para facilitar a análise no dashboard:

- Período de experiência
- Até 1 ano
- De 1 a 3 anos
- Mais de 3 anos

---

## ETL com Apache Hop

O processo de ETL é dividido entre pipelines e workflows.

### Pipeline

Arquivo:

```text
/files/pipelines/turnover.hpl
```

Responsável pelas principais transformações dos dados, incluindo:

- leitura das fontes;
- padronização dos campos;
- tratamento de caracteres;
- cálculo do tempo de empresa;
- conversão do tempo em dias, meses e anos;
- classificação do tempo de casa;
- classificação do tipo de desligamento;
- preparação dos dados para carga no PostgreSQL.

### Workflow

Arquivo:

```text
/files/workflows/rh_rotina.hwf
```

O workflow controla a execução da rotina de ETL e organiza as etapas do processo.

Entre as responsabilidades estão:

- execução da pipeline;
- validações;
- controle da rotina;
- tratamento de falhas;
- execução das etapas necessárias para atualização da tabela `fato_desligamentos`.

---

## Automação com Jenkins

O Jenkins é utilizado para automatizar a execução do processo de ETL.

A execução acontece dentro do ambiente Docker, utilizando o container do Apache Hop.

Exemplo de execução:

```bash
docker exec -it hop-web /opt/hop/hop-run.sh \
  --file=/files/workflows/rh_rotina.hwf \
  --runconfig=local
```

Com isso, o processo pode ser executado sem a necessidade de abrir o Apache Hop manualmente.

---

# Power BI

## Modelo de dados

Para o dashboard foi utilizado um modelo dimensional simples, baseado em uma tabela fato e uma dimensão de calendário.

A tabela principal é:

```text
fato_desligamentos
```

E a dimensão utilizada para as análises temporais:

```text
d_calendario
```

A relação entre as tabelas segue o padrão:

```text
d_calendario (1) ──────────── (*) fato_desligamentos
```

### Dimensão calendário

A tabela de calendário foi criada em DAX:

```DAX
d_calendario =
ADDCOLUMNS (
    CALENDAR(DATE(2022, 1, 1), DATE(2026, 12, 31)),
    "Ano", YEAR([Date]),
    "Mês", FORMAT([Date], "mmmm"),
    "Mês Num", MONTH([Date]),
    "AnoMês", FORMAT([Date], "YYYY-MM"),
    "Trimestre", "T" & FORMAT([Date], "Q")
)
```

A dimensão permite trabalhar com análises mensais, anuais e comparações entre períodos.

---

## Principais medidas DAX

### Total de desligamentos

```DAX
Total Desligamentos =
COUNTROWS(fato_desligamentos)
```

### Tempo médio de empresa

```DAX
Tempo Médio Casa (Meses) =
AVERAGE(fato_desligamentos[meses_empresa])
```

### Percentual de desligamentos voluntários

```DAX
% Voluntarios =
DIVIDE(
    CALCULATE(
        [Total Desligamentos],
        fato_desligamentos[tipo_desligamento] = "voluntario"
    ),
    [Total Desligamentos],
    0
)
```

---

## Indicadores de turnover

Para este projeto foi considerada uma base fixa de **500 colaboradores** para o cálculo dos indicadores de turnover.

### Turnover mensal

```DAX
Turnover Mensal % =
VAR HeadcountFixo = 500
RETURN
DIVIDE(
    [Total Desligamentos],
    HeadcountFixo,
    0
)
```

### Turnover anualizado

```DAX
Turnover Anualizado % =
VAR HeadcountFixo = 500
RETURN
DIVIDE(
    [Total Desligamentos] * 12,
    HeadcountFixo,
    0
)
```

---

## Análise temporal

Também foram criadas medidas para comparar os desligamentos entre períodos.

### Desligamentos no mês anterior

```DAX
Desligamentos Mês Anterior =
CALCULATE(
    [Total Desligamentos],
    PARALLELPERIOD(
        d_calendario[Date],
        -1,
        MONTH
    )
)
```

### Variação mensal

```DAX
Variacao MoM % =
VAR Atual = [Total Desligamentos]
VAR Anterior = [Desligamentos Mês Anterior]
RETURN
DIVIDE(
    Atual - Anterior,
    Anterior,
    0
)
```

Essas medidas permitem acompanhar a evolução dos desligamentos e identificar aumentos ou reduções em relação ao mês anterior.

---

# Dashboard

O dashboard foi estruturado para permitir uma visão geral dos desligamentos e, ao mesmo tempo, possibilitar o aprofundamento nas informações.

## Visão executiva

A primeira camada apresenta os principais indicadores:

- Total de desligamentos;
- Turnover;
- Tempo médio de permanência;
- Percentual de desligamentos voluntários;
- Variação em relação ao mês anterior.

A ideia é permitir uma leitura rápida do cenário antes de partir para as análises mais detalhadas.

## Motivos dos desligamentos

Os motivos são apresentados de forma comparativa para identificar quais fatores possuem maior participação no volume de saídas.

Entre as categorias analisadas estão, por exemplo:

- Pedido do colaborador;
- Reestruturação;
- Performance;
- Cultura / Fit.

## Segmento

Os desligamentos também podem ser analisados de acordo com o segmento do colaborador, como:

- Administrativo;
- Operacional;
- Pedagógico.

Isso permite identificar onde está concentrado o maior volume de saídas.

## Tempo de casa

Outra análise importante é a distribuição dos desligamentos por tempo de permanência.

As categorias utilizadas são:

- Período de experiência;
- Até 1 ano;
- De 1 a 3 anos;
- Mais de 3 anos.

Essa visão ajuda a identificar, por exemplo, se existe uma concentração de desligamentos entre colaboradores recém-admitidos ou entre pessoas com maior tempo de empresa.

---

## Tooltip de detalhamento

Foi utilizado um tooltip personalizado no Power BI para permitir o detalhamento das informações.

Ao passar o mouse sobre determinados elementos dos gráficos, é possível visualizar informações adicionais relacionadas ao contexto selecionado, incluindo os registros das saídas correspondentes àquele período ou categoria.

Isso evita a necessidade de colocar todas as informações diretamente na página principal do dashboard.

---

# Como executar o projeto

## Pré-requisitos

Para executar o projeto localmente, é necessário ter instalado:

- Docker;
- Docker Compose;
- Git;
- Power BI Desktop.

## 1. Clonar o repositório

```bash
git clone https://github.com/seu-usuario/seu-repositorio.git

cd seu-repositorio
```

## 2. Subir os containers

```bash
docker-compose up -d
```

Esse comando inicia os serviços necessários para o ambiente, incluindo o PostgreSQL e o Apache Hop.

## 3. Executar o ETL

Com os containers em execução:

```bash
docker exec -it hop-web /opt/hop/hop-run.sh \
  --file=/files/workflows/rh_rotina.hwf \
  --runconfig=local
```

Após a execução, os dados tratados estarão disponíveis no PostgreSQL.

## 4. Abrir o Power BI

Abra o arquivo `.pbix` localizado na pasta:

```text
/pbix
```

Configure a conexão com o PostgreSQL utilizando as credenciais definidas no ambiente.

Por padrão, a conexão local utiliza:

```text
Host: localhost
Port: 5432
```

As demais informações de acesso devem ser ajustadas de acordo com o `docker-compose` utilizado no projeto.

---

# Estrutura do projeto

Uma possível organização dos arquivos é:

```text
/
├── docker-compose.yml
│
├── files/
│   ├── pipelines/
│   │   └── turnover.hpl
│   │
│   └── workflows/
│       └── rh_rotina.hwf
│
├── pbix/
│   └── dashboard_turnover.pbix
│
└── README.md
```

---

# Objetivo do projeto

O objetivo principal é transformar dados operacionais de RH em informações que possam ser utilizadas na análise de turnover.

Além do dashboard, o projeto busca demonstrar na prática a construção de um fluxo completo de dados:

```text
Dados brutos
    ↓
ETL
    ↓
Tratamento e regras de negócio
    ↓
PostgreSQL
    ↓
Modelagem dimensional
    ↓
Medidas DAX
    ↓
Dashboard
```

Dessa forma, o projeto não fica restrito à construção do relatório no Power BI. O processo contempla também a preparação, armazenamento e atualização dos dados que alimentam os indicadores.

---

# Autor

**Felipe Yuiti Kawaji**

Analista de Dados | People Analytics

Projeto desenvolvido com foco em Engenharia de Dados, automação de processos e Business Intelligence aplicado à área de Recursos Humanos.