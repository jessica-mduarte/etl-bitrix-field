# Banco de Dados - PostgreSQL

## Objetivo

Este diretório contém os scripts SQL responsáveis pela criação e configuração do banco de dados do projeto Extrator de Relatórios Bitrix24 e Field Control. O banco utiliza a arquitetura Medallion, organizada em quatro schemas: `bronze, silver, gold` e `ctrl`. As permissões de acesso são configuradas separadamente, seguindo o princípio do menor privilégio. 

## Estrutura de diretórios

```
db/
├── README.md
├── bronze/
│   └── 01_create_bitrix_raw.sql
├── silver/
│   ├── 01_create_dimensions.sql
│   └── 02_create_cards.sql
├── gold/
│   └── 01_create_views.sql
├── ctrl/
│   └── 01_create_control_tables.sql
└── roles/
    └── 01_create_permissions.sql
```

Cada diretório organiza os scripts SQL de uma camada ou responsabilidade. Esses diretórios são apenas uma organização dos arquivos do projeto, visto que **os schemas correspondentes são criados dentro do PostgreSQL**.

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




## Ordem de criação

Os scripts devem ser executados no PostgreSQL na seguinte ordem lógica:

1. **Schemas e tabelas de controle**: criar os schemas e as estruturas necessárias ao acompanhamento das execuções.
2. **Bronze**: criar as tabelas destinadas ao armazenamento bruto.
3. **Silver**: criar as dimensões, os relacionamentos e as tabelas de cards e ordens de serviço.
4. **Gold**: criar as views após a existência das tabelas Silver utilizadas nas consultas.
5. **Permissões**: cnofigurar as roles e conceder os privilégios necessários para o ETL e consumo de dados.

A ordem física dos arquivos poderá ser ajustada conforme as dependências entre os scripts. A execução dos scripts deve ser validada antes de iniciar o ETL.

## Convenções

- Os nomes dos schemas e das tabelas devem ser consistentes com os utilizados pelo código Python.
- As tabelas Silver devem possuir chaves primárias adequadas à identificação dos registros. 
- Os scripts devem ser organizados por camada e numerados para indicar a ordem de execução.
- As operações de carga devem permitir reprocessamento sem gerar duplicatas.
- As credenciais de conexão não devem ser armazenadas nos scripts SQL nem versionadas no repositório.

## Permissões de acesso

O projeto utiliza credenciais separadas para a execução do ETL e para o consumo de dados. 

- **Role do ETL**: responsável por executar as operações necessárias de leitura e escrita nas camadas de processamento.
- **Role de BI**: possui acesso somente de leitura ás views da camada Gold.

As permissões definitivas serão configuradas nos scripts de `roles/` e deverão refletir as operações efetivamente necessárias para cada função.

Para detalhes sobre o fluxo completo de dados, as decisões de arquitetura e as configurações do projeto, consulte o [`README.md`](https://github.com/jessica-mduarte/etl-bitrix-field/blob/main/README.md) principal na raíz do repositório.