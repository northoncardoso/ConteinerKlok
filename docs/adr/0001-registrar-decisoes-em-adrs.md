# ADR 0001: Registrar as decisoes de arquitetura em ADRs

Status: aceito
Data: 2026-10-13

## Contexto

O ConteinerKlok e um projeto novo, com varias escolhas estruturais feitas antes
de qualquer codigo: ferramenta de orquestracao, formato do AppHost, banco de
dados, estrategia de teste e escopo de entrega. Sem registro, esse racional se
perde e cada decisao vira uma discussao repetida.

## Decisao

Registrar cada decisao estrutural em um ADR (Architecture Decision Record), no
formato enxuto de Michael Nygard: titulo, status, contexto, decisao e
consequencias. Os arquivos ficam em `docs/adr/`, numerados em ordem.

Um ADR aceito nao e reescrito. Quando uma decisao muda, cria-se um novo ADR que
substitui o anterior e o status do antigo vira "substituido pelo ADR N".

## Consequencias

- O racional fica visivel para quem chega depois, incluindo o proprio autor.
- Exige disciplina de escrever um ADR a cada decisao relevante.
- Fornece material pronto para o portifolio e para futuras entrevistas.
