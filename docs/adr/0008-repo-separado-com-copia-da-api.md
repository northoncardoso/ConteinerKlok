# ADR 0008: Repositorio separado, com copia da API

Status: aceito
Data: 2026-10-13

## Contexto

O trabalho de containerizacao poderia morar dentro do repo Klok, ou referenciar
o codigo do Klok de fora (submodule ou clone no build). O autor pediu um projeto
novo, fora do Klok, sem poluir aquele repositorio com .NET e Aspire.

## Decisao

O ConteinerKlok e um repositorio proprio e autocontido. Ele contem uma copia da
`klok-api`, ja portada para Postgres, mais o AppHost, o Dockerfile, os testes e a
documentacao. Nao ha dependencia do repo Klok.

## Consequencias

- Positivo: o projeto roda sozinho, serve de portifolio e nao adiciona .NET nem
  Aspire ao Klok.
- Negativo: existem duas copias da API (a do Klok e a daqui). Se a API do Klok
  mudar, a copia precisa ser atualizada a mao.
- As mudancas de banco (Postgres) valem so para este repo; o Klok segue em
  SQLite.
