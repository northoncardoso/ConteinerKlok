# ADR 0007: Publicar docker compose, sem deploy na nuvem

Status: aceito
Data: 2026-10-13

## Contexto

O escopo de entrega foi definido com o autor: local mais um artefato de publish.
Deploy em nuvem (por exemplo, Azure Container Apps) exigiria conta, credenciais
e um conjunto novo de decisoes.

## Decisao

Cobrir o ambiente local com `aspire start` e gerar o artefato de container com
`aspire publish`, que produz um docker compose. Deploy em nuvem fica fora deste
projeto, para um trabalho futuro.

## Consequencias

- Positivo: um artefato conteinizado reutilizavel e independente da Aspire, e um
  escopo que cabe em um plano so.
- O compose gerado vira o ponto de partida de um deploy futuro, mas nao e
  publicado nem hospedado aqui.
