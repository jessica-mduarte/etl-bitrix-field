# Extrator de Relatórios Bitrix24 & Field Control

Projeto de dados que extrai informações de CRM do **Bitrix24**(de forma incremental) e ordens de serviço do **Field Control** (por janelas de meses), as organiza em arquitetura medallion em um banco **PostgreSQL** e alimenta o Power BI e relatórios XLSX. 

**Status**: trabalho em progresso. Este documento descreve a arquitetura-alvo, quais foram as decisões tomadas e a responsabilidade de cada camada. Este README.md será atualizado e resumido ao longo do projeto, quando os arquivos .md das demais pastas forem apropriamente atualizadas.

## Sumário 

1. [Objetivo]
2. [Visão Geral]
2. [Stacks]
3. [Decisões de arquitetura]
4. [Camada Bronze]
5. [Camada Silver]
6. [Camada Gold]
7. [Estrutura do projeto]
8. [Controle, execução e falhas]

## Objetivo

- Guardar o histórico dos resultados da empresa. O B24 mantém somente o estado atual de cada card. 
- Dados de manutenções preventivas realizadas por mês serão extraídas a partir do Field Control. 
- Permitir filtros **mensais e anuais** sem depender diretamente da API. 
- Gerar arquivos em .xlsx de resumo claros para envio à gestão. 
- Garantir que Power BI e relatórios leiam da **mesma fonte**.
- Escalar com o cresimento do número de cards sem reler a base inteira a cada execução. 

## Visão geral

![Fluxo de dados](images/fluxo.png)

| Schema | Conteúdo | Quem escreve | Quem lê |
|---|---|---|---|
| `bronze` | Dados brutos devolvidos pela API | Python (extract) | Python (silver) |
| `silver` | Estado atual de cards e OS, limpos e validados | Python/pandas | Python, SQL(gold) |
| `gold` | Visões de negócio prontas para consumo | SQL(views) | Power BI e Excel |
| `ctrl` | Controle de execução e qualidade | Python | Python |

Possuímos um único banco e vários schemas. A gold é construída por SQL direto sobre a silver, e as permissões são separadas por schema. 

## Stacks e configuração

| Ferramenta | Propósito |
|---|---|
| Python 3.10+ | Orquestração do ETL |
| pandas | Limpeza e padronização (camadas bronze e silver) |
| SQLAlchemy + psycopg | Conexão com PostgreSQL |
| PostgreSQL | Armazenamento das camadas |
| Power BI | Dashboards, via conector PostgreSQL |

```
git clone <url-do-repositorio>
cd projeto-bi-bitrix

python -m venv .venv
source .venv/bin/activate        # Windows: .venv\Scripts\activate
pip install -r requirements.txt

cp .env.example .env             # preencha os valores reais
```

| Variável | Descrição |
|---|---| 
| `BITRIX_WEBHOOK_URL` | URL do webhook do Bitrix24 |
| `DB_HOST`, `DB_PORT`, `DB_NAME` | Conexão com o PostgreSQL |
| `DB_SSLMODE` | Ex.: `require` |
| `DB_ETL_USER`, `DB_ETL_PASSWORD` | Role do ETL |
| `DB_BI_USER`, `DB_BI_PASSWORD` | Role somente da leitura da gold |
| `INCREMENTAL_OVERLAP_MINUTES` | Margem de sobreposição da carga incremental |
| `FATURAMENTO_JANELA_MESES` | Quantos meses recentes a gold deve recalcular a cada execução |

Agendamento (cron), logs e restauração: `docs/OPERACAO.md`.


## Decisões de arquitetura

| Decisão | Motivo | Rever caso: |
|---|---|---|
| **Arquitetura medallion** | Separa o que veio da API (bronze), o que foi tratado (silver) e o que a gestão deve consumir (gold) | Se a manutenção das camadas ficar mais trabalhosa que o benefício |
| **Bitrix24 em carga incremental** | O volume de cards tende a crescer, e reler todo dia todo o conteúdo extraído aumenta o tempo e consumo da API sem necessidade | Se o incremental ficar instável, uma carga completa diária é válida enquanto o volume for baixo |
| **Field Control por janela de meses, sem marca** | O volume de OS com a tag é menor. Refazer a janela inteira a cada execução captura mudanças de status, tags removidas e OS canceladas, dispendando a reconciliação de exclusões para essa fonte | Se o volume de OS crescer muito |
| **Bronze genérica (`raw_items`) em uma única tabela com JSONB** | Uma única rotina de carga serve para todas as fontes, e campos novos do Bitrix não quebram a carga. A tabela cresce somente com número de itens, não com as alterações | Se for necesspario auditar versões antigas, passar a gravar cada versão como uma nova linha |
| **Carga de reconciliação semanal só de ids(Bitrix)** | A carga incremental não mostra itens excluídos no Bitrix. Comparar ids existentes com silver detecta exclusões sem baixar tudo | Se o Bitrix passa a oferecer uma lista de exclusões confiável |
| **Silver com pandas, gold com SQL** | Limpeza de JSON e de campos personalizados é mais simples em python, enquanto agregações por mês e ano são mais rápidas e claras no banco | Se a limpeza ficar toda no SQL, simplificar |
| **Gold como views comuns, sem materialização** | Com o volume atual, views respondem rápido e nunca ficam desatualizadas. Materializa-se depois, se necessário | Caso uma consulta ao Power BI fique lenta |
| **Estados por camada em `ctrl`** | Cada lote tem status separado por camada. Caso a silver falhe, pode reprocessar a partir da camada bronze sem chamar a API | - |
| **Sem orquestrador(Airflow, etc)** | Usaremos o cron que já resolve o agendamento. Menos peças para manter | Se surgirem dependências entre muitos jobs |
| **Menor privilégio por role** | O Power BI e Xlsx. lêem somente a gold. Quem escreve é a role do ETL | - |

## Camada Bronze

**Responsabilidade**

Guarda o JSON mais recente de cada item, exatamente como a API devolveu. É um espelho cru que permite reconstruir a silver sem chamar a API novamente. Guarda todos os campos, inclusive aqueles que o projeto ainda não usa.

| Regra | Descrição |
|---|---|
| Uma linha por item | A chave é (`source, item_id`). A tabela cresce com o número de itens, não com as alterações |
| Margem inofensiva | Um item que volta duas vezes por causa da margem é apenas regravado com o mesmo conteúdo |
| Sem restrições | Sem FK e sem tipos além de `jsonb`, para garantir que a carga nunca falhará por dependência |
| Carga inicial | O modo `full` extrai sem filtro de data e grava todos os itens |

**Estrutura** 
```
CREATE TABLE bronze.bitrix_raw (
    source         text        NOT NULL,   -- 'comercial', 'edv', 'instalacoes', 'operacoes', 'users', 'teams', 'stages'
    item_id        bigint      NOT NULL,   -- id do registro no Bitrix
    source_method  text        NOT NULL,   -- 'crm.item.list', 'crm.deal.list', 'user.get', ...
    entity_type_id int,                    -- NULL para usuários e times
    payload        jsonb       NOT NULL,   -- resposta completa, sem alterar
    batch_id       uuid        NOT NULL,   -- último lote que gravou este item
    extracted_at   timestamptz NOT NULL DEFAULT now(),
    PRIMARY KEY (source, item_id)
);

CREATE INDEX ON bronze.bitrix_raw (batch_id);
```

**Passo a passo: Bitrix24 (modo incremental)** 
1. **Ler a marca** da fonte em `ctrl.marcas`(data/hora da última extração bem-sucedida).
2. **Montar filtro de modificação**: itens alterados depois de `marca - margem`.
3. **Extrair** da API com paginação e respeito ao llimite de requisições, sem `select` restritivo, para guardar todos os campos. 
4. **Gravar com upsert** (`INSERT ... ON CONFLICT (source, item_id) DO UPDATE`), atualizando `payload`, `batch_id` e `extracted_at`.
5. **Registrar** `bronze_status = OK` em `ctrl.execucoes` e avançar a marca da fonte. 

**Passo a passo: Field Control (por janela)**
1.**Calcula** a janela: do primeiro dia do mês mai antigo da janela (FIELD_JANELA_MESES) até hoje.
2. **Extrai** as Ordens de Serviço que possuem a tag de manutenção preventiva. O filtro por tag e data é aplicado na API (quando ela suportar), ou no Python (quando não suportar).
3. **Grava**  com upsert em `bronze.raw_items` com `source = 'field_os`.
4. **Registra** `bronze_status = OK`. Não há marca a avançar, a próxima execução refaz a janela inteira.

## Camada Silver

**Responsabilidade**

Transformar os dados brutos em dados confiáveis e consultáveis, ou seja, tipados, padronizados, sem duplicatas e com relacionamentos. Guarda somente o estado atual de cada card. É onde mora a qualidade dos dados, e é a base do cálculo do faturamento.

| Regra | Descrição |
|---|---|
| Uma linha por entidade | Chave primária por tabela (`card_id`, ou `entity_type_id` + `card_id` se unificar fontes) |
| Idempotência | Reprocessar um lote produz o mesmo resultado |
| Nomes padronizados | `created_at`, `updated_at`, `moved_at`, `opened_at`, `closed_at`, `deadline_at` |

**Tabelas previstas**
| Tabela | Conteúdo |
|---|---|
| `silver.dim_users` | Usuários (id, nome, cargo, departamento, `is_active`). Usuários desligados não são apagados. |
| `silver.user_goals` |Metas por usuário anuais e mensais |
| `silver.dim_teams` | Times (Silver e Black) | 
| `silver.dim_stages` | Etapas: código, nome, funil e semântica (em andamento, ganho ou perdido)
| `silver. comercial_cards`, `edv_cards`, `instalacoes_cards`, `operacoes_cards` | Estado atual de cada card, padronizado e limpo |
| `silver.field_os` | Ordens de serviço da tag: id da OS, tag, status, data de conclusão e `is_deleted`. |

Colunas de auditoria nas tabelas de cards: `batch_id`, `loaded_at` e `is_deleted`.

**Passo a passo**
1. **Seleciona** os lotes com `silver.status` pendente em `ctrl.execucoes`.
2. **Lê** da camada bronze, em blocos(`chunksize`), as linhas desses lotes. Sem carregar a camada bronze inteira na memória.
3. **Normaliza** o JSON com pandas e aplica o mapeamento `config/field_mapping`: renomear, converter tipos, traduzir ids de lista para texto e tratar campos personalizados.
4. **Valida** ids duplicados, nulos em campos obrigatórios, datas, ids de usuário ou etapas sem correspondência nas dimensões.
5. **Upsert** do estado atual com `INSERT ... ON CONFLICT DO UPDATE`. Reprocessar o mesmo lote não duplica nada.
6. **Field Control**: as OS que estavam na silver dentro da janela e não voltaram da chamada da API (porque foram canceladas, excluídas ou perderam a tag) recebem `is_deleted = true`.
6. **Registra** `silver_status = OK` e as contagens em `ctrl_execucoes`.

**Reconciliação de exclusões do Bitrix (modo `reconcile`)**

1. Extrai só os ids de cada fonte no Bitrix (`select[]=id`).
2. Compara com os ids ativos da silver.
3. Ids que não existem mais no Bitrix recebem `is_deleted = true`. Nada é apagado fisicamente.

## Camada Gold
 **Responsabilidade** 

 Entregar todos os dados, prontos para consumo. É a única camada que o Power BI e XLSXs enxergam, e guarda o único histórico do projeto. 

| Regra | Descrição |
|---|---|
| Não modifica dados | Todo o processo de limpeza e padronização já ocorreu na camada Silver. |
| Não contém dados além do necessário | Armazena somente os dados que devem ser exibidos no Power BI e XLSX. |
| Janela de atualização | A cada execução, a camada recalcula apenas os meses mais recentes. Meses fora da janela ficam com `fechado = true` e não são tocados. |
| Carga inicial | Calcula todos os meses a partir da camada silver e fecha os que estiverem fora da janela. |

## Estrutura do projeto

```
projeto-bi/
│
├── README.md
├── .env.example
├── .gitignore
├── requirements.txt
│
├── config/
│   ├── sources.yml --define quais fontes consultar, quais métodos de API utilizar e quais campos de data ou filtros considerar
│   └── field_mapping.yml --define como renomear, interpretar e converter os campos recebidos para o formato da Silver
│
├── etl/
│   ├── extract/    --consulta as APIs
│   │   ├── bitrix/
│   │   │   ├── __init__.py
│   │   │   ├── users.py
│   │   │   └── crm.py
│   │   └── field_control/
│   │       ├── __init__.py
│   │       └── work_orders.py
│   │
│   ├── bronze_to_silver/   --limpa e padroniza os dados com pandas
│   │   ├── __init__.py
│   │   ├── users.py
│   │   └── comercial_cards.py
│   │
│   ├── reconcile/      --identifica registros que deixaram de existir no Bitrix
│   │   ├── __init__.py
│   │   └── bitrix.py
│   │
│   ├── gold/       --executa o processamento necessário para disponibilizar indicadores
│   │   ├── __init__.py
│   │   └── monthly_metrics.py
│   │
│   ├── common/     --concentra funções compartilhadas
│   │   ├── __init__.py
│   │   ├── database.py
│   │   ├── logging_config.py
│   │   └── batch_control.py
│   │
│   └── main.py     --coordena a execução das etapas
│
├── db/
│   ├── bronze/     --tabelas para os dados brutos  
│   │   └── 01_create_bitrix_raw.sql
│   ├── silver/     --tabelas tratadas e relacionamentos
│   │   ├── 01_create_dimensions.sql
│   │   └── 02_create_cards.sql
│   ├── gold/       --views para Power BI e relatórios
│   │   └── 01_create_views.sql
│   ├── ctrl/       --tabelas de controle de execução e lotes
│   │   └── 01_create_control_tables.sql
│   ├── roles/      --permissões dos usuários do banco
│   │   └── 01_create_permissions.sql
│   └── README.md
│
├── reports/
│   ├── README.md       --documenta os relatórios e indicadores
│   ├── generate_reports.py     --consulta os dados e gera os arquivos .xlsx.
│   └── output/     --recebe os arquivos gerados
│       └── .gitkeep
│
├── powerbi/
│   └── README.md       --explica a conexão com PostgreSQL, medidas DAX, calendário e atualização dos dados
│
├── docs/
│   ├── ARQUITETURA.md      --explica as camadas, o fluxo e decisões da arquitetura
│   └── OPERACAO.md     --explica como executar, agendar, diagnosticar falhas e recuperar o processamento
│
└── tests/      --verificam se a extração, as transformações e reconciliação funcionam conforme esperado, sem precisar executar todo o ETL a cada teste
    ├── test_extraction.py
    ├── test_transformation.py
    └── test_reconciliation.py
```