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

### bronze

Armazena os dados brutos retornados pelas APIs do Bitrix24 e do Field em formato JSONB.

- Preserva os campos retornados pelas fontes sem exigir um mapeamento prévio de todos eles.
- Permite reconstruir a camada Silver sem consultar novamente a API, utilizando o conteúdo bruto mais recente armazenado.
- Utiliza operações de upsert para atualizar os registros existentes.
- Não utiliza chaves estrangeiras para não criar dependências desnecessárias durante a ingestão.

### silver

Armazena os dados padronizados e tratados, representando o estado atual dos registros.

- Converte os dados brutos para tipos adequados ao PostgreSQL.
- Padroniza nomes de campos e trata campos personalizados.
- Mantém relacionamentos entre entidades por meio de chaves primárias e estrangeiras, quando aplicável. 
- Utiliza upsert para permitir o reprocessamento idempotente dos dados.
- Mantém registros que deixaram de existir na origem marcados como excluídos logicamente, quando aplicável. 

### gold

Disponibiliza os dados preparados para consumo pelo Power BI e pelos relatórios Excel.

- Contém views SQL com indicadores e informações consolidadas para a gestão. 
- Consulta os dados tratados da Silver.
- É a única camada de dados acessada pelas ferramentas de consumo. 
- Utiliza views comuns inicialmente, sem materialização. 

### ctrl 

Armazena informações de controle da execução do ETL.

- Registra os lotes processados e o status de cada camada.
- Mantém as marcas utilizadas para controlar a extração incremental.
- Permite identificar falhas e reprocessar dados a partir da Bronze, quando possível. 
- Armazena informações necessárias ao monitoramento da execução.

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