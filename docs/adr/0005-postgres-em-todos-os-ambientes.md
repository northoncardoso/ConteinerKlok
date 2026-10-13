# ADR 0005: Usar Postgres em todos os ambientes

Status: aceito
Data: 2026-10-13

## Contexto

A API do Klok usa `node:sqlite` com uma API sincrona (`DatabaseSync`,
`.prepare().run()`), e os testes rodam em banco `:memory:`. O objetivo do
ConteinerKlok e orquestrar um servico de banco de verdade, o que pede Postgres.

## Decisao

Substituir SQLite por Postgres em desenvolvimento, testes e container. Nao
mantemos dois drivers.

## Consequencias

- A camada de banco vira assincrona (`pg` com `Pool`), o que se propaga para
  todos os controllers e middlewares.
- O esquema muda de dialeto: `AUTOINCREMENT` vira `GENERATED ALWAYS AS IDENTITY`,
  `lastInsertRowid` vira `RETURNING id`, `usado` vira `boolean` e as datas viram
  `timestamptz`.
- O mecanismo de migracao sai de `PRAGMA user_version` para uma tabela
  `schema_migrations`.
- Positivo: paridade total entre dev, teste e container, e nenhum dialeto
  escondido.
- O CI passa a depender de um Postgres para rodar os testes.

## Alternativas descartadas

- Dois drivers (SQLite em teste, Postgres em producao): manteria os testes
  rapidos, mas os testes deixariam de exercitar o banco de producao.
- Adiar o Postgres e so containerizar com SQLite: menor esforco, porem nao
  cumpre o objetivo de orquestrar um banco de verdade.
