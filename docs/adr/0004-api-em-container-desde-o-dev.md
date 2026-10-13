# ADR 0004: Rodar a API em container desde o desenvolvimento

Status: aceito
Data: 2026-10-13

## Contexto

O Aspire pode rodar um app Node de duas formas: direto pelo runtime
(`AddNpmApp`, com watch) ou como imagem construida a partir de um Dockerfile
(`AddDockerfile`). O objetivo do projeto e demonstrar conteinerizacao real.

## Decisao

Rodar a API como container desde o desenvolvimento, via `AddDockerfile`. O
mesmo caminho vale para dev e para o publish.

## Consequencias

- Positivo: fidelidade ao que roda em producao, e o Dockerfile e exercitado o
  tempo todo.
- Negativo: cada mudanca no codigo exige reconstruir a imagem, sem watch quente.
  Para iterar rapido, o desenvolvedor pode rodar o Node direto fora do Aspire
  quando quiser.
