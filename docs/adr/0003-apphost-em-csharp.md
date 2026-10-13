# ADR 0003: Escrever o AppHost em C#

Status: aceito
Data: 2026-10-13

## Contexto

O Aspire aceita o AppHost em tres formatos: projeto C#, arquivo C# unico
(`apphost.cs`) e TypeScript (`apphost.mts`). O restante do repositorio e Node.

## Decisao

Usar um projeto C# (.NET 10) para o AppHost, em
`aspire/ConteinerKlok.AppHost/`.

## Consequencias

- Positivo: e o formato com melhor suporte, documentacao e ferramental, e
  aprofunda o .NET do autor, que ja tem o SDK instalado.
- Negativo: adiciona .NET a um repositorio dominado por Node.

## Alternativas descartadas

- `apphost.cs` file-based: menos estrutura, mas suporte e exemplos mais escassos.
- TypeScript: mantem uma linguagem so, porem e o formato mais novo e com menos
  material de referencia.
