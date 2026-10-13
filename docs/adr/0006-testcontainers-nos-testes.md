# ADR 0006: Usar testcontainers nos testes de integracao

Status: aceito
Data: 2026-10-13

## Contexto

Com o Postgres em todos os ambientes, os testes precisam de um Postgres real e
isolado. As opcoes eram: testcontainers (sobe container efemero por execucao),
um Postgres fixo via compose ou servico no CI, ou um banco compartilhado com
limpeza manual.

## Decisao

Usar `@testcontainers/postgresql`. Cada execucao de `npm test` sobe um Postgres
efemero, roda as migracoes em um banco isolado e derruba o container no fim.

## Consequencias

- Positivo: isolamento total, nenhum estado entre execucoes, e o mesmo comando
  funciona local e no CI (ubuntu tem Docker).
- Negativo: exige Docker para testar, e cada execucao paga o tempo de subir o
  container e, na primeira vez, de baixar a imagem.
- Adiciona uma dependencia de desenvolvimento.
