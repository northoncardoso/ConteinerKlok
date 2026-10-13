# ConteinerKlok

Containerizacao da API do [Klok](https://github.com/northoncardoso/Klok) com
[Aspire](https://aspire.dev) e Postgres.

Projeto de estudo e portifolio: pega o backend real de gestao de ponto do Klok,
porta o banco de SQLite para Postgres, empacota a API em container e sobe tudo
orquestrado pelo Aspire.

## Objetivo

- Servir de cobaia para aprender Aspire e conteinerizacao com um servico real.
- Produzir um artefato reutilizavel: `aspire publish` gera um docker-compose.
- Manter o comportamento da API do Klok.

## Estado

Em planejamento. O design fica em `docs/superpowers/specs/` e as decisoes de
arquitetura em `docs/adr/`.

## Estrutura prevista

```
api/                      copia da klok-api portada para Postgres
aspire/ConteinerKlok.AppHost/   AppHost em C#
docs/adr/                 decisoes de arquitetura
docs/superpowers/specs/   design
docs/superpowers/plans/   plano de implementacao
```

## Como vai rodar

`aspire start` sobe a API em container e o Postgres. Os detalhes estao no design
em `docs/superpowers/specs/`.

## Fora de escopo

Deploy na nuvem e o app mobile (Expo) nos containers.
