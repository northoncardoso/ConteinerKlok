# ADR 0002: Usar o Aspire como orquestrador local

Status: aceito
Data: 2026-10-13

## Contexto

Precisamos subir a API e o Postgres juntos, de forma reproduzivel, com
descoberta de servico, variaveis de ambiente e um painel de observabilidade.
As alternativas seriam um docker compose escrito a mao ou scripts de shell.

## Decisao

Usar o Aspire com um AppHost, que descreve os recursos em codigo. O AppHost sobe
a API (em container) e o Postgres, injeta a connection string e expoe o
dashboard.

## Consequencias

- Positivo: ambiente reproduzivel, dashboard pronto, e o mesmo modelo gera o
  docker compose via `aspire publish`.
- Exige o SDK .NET 10 e o Aspire CLI instalados na maquina.
- Adiciona uma segunda stack (a Aspire) a um projeto de base Node.
