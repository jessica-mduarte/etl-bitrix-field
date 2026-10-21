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

#### bronze

Armazena os dados brutos retornados pelas APIs do Bitrix24 e do Field em formato JSONB.

- Preserva os campos retornados pelas fontes sem exigir um mapeamento prévio de todos eles.
- Permite reconstruir a camada Silver sem consultar novamente a API, utilizando o conteúdo bruto mais recente armazenado.
- Utiliza operações de upsert para atualizar os registros existentes.