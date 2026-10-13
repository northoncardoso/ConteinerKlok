# ConteinerKlok: containerizar a API do Klok com Aspire e Postgres

Data: 2026-10-13
Status: design aprovado (aguardando revisao da spec escrita)

## Contexto e objetivo

O ConteinerKlok e um projeto novo, separado do repo Klok, criado para servir de
cobaia pratica no aprendizado de Aspire e conteinerizacao usando um servico real.
O material de partida e o backend de gestao de ponto do Klok (`klok-api`), uma API
Node.js e Express.

Objetivo: subir a API em container e um Postgres em container, orquestrados por
um AppHost do Aspire, mantendo o comportamento atual da API, e gerar um
docker compose via `aspire publish`.

Sucesso:
- `aspire start` sobe a API e o Postgres, e a API responde como hoje.
- `npm test` passa, contra um Postgres efemero.
- `aspire publish` gera um docker compose executavel.
- O repositorio e autocontido e defensavel como peca de portifolio.

## Nao objetivos

- Deploy na nuvem.
- O app mobile (Expo) dentro de container.
- Manter compatibilidade com o repo Klok ou sincronizar codigo com ele.

## Arquitetura

- Repositorio ConteinerKlok, autocontido, com uma copia da `klok-api`.
- AppHost em C# (`aspire/ConteinerKlok.AppHost/`), descrevendo dois recursos: a
  API (imagem construida a partir de um Dockerfile) e o Postgres (com volume).
- A API recebe a connection string do Postgres por variavel de ambiente,
  injetada pela referencia do AppHost. Segredos (`JWT_SECRET`, SMTP, client id do
  Google) entram por parameters do AppHost, nunca no repo.
- `aspire start` para o ambiente local; `aspire publish` para o docker compose.
- O app Expo fica fora, apenas apontando para a URL da API.

## Estrutura do repositorio (prevista)

```
api/                            copia da klok-api (portada para Postgres)
  Dockerfile
  .dockerignore
  src/... (ou a estrutura atual de arquivos)
  test/
aspire/ConteinerKlok.AppHost/   AppHost em C#
docs/adr/                       decisoes de arquitetura
docs/superpowers/specs/         este design
docs/superpowers/plans/         plano de implementacao
.github/workflows/ci.yml        testes da API
```

## Camada de banco (SQLite para Postgres)

O `db.js` atual usa `node:sqlite` com API sincrona. A troca para Postgres tem
quatro frentes:

1. Driver: sai `DatabaseSync`, entra `pg` com um `Pool`. Toda funcao do banco
   passa a devolver `Promise`, e os controllers e middlewares ganham `await`.
2. Esquema (dialeto):
   - `INTEGER PRIMARY KEY AUTOINCREMENT` vira `GENERATED ALWAYS AS IDENTITY`.
   - `lastInsertRowid` vira `RETURNING id`.
   - `usado INTEGER` vira `boolean`.
   - datas em texto viram `timestamptz`.
   - os indices unicos parciais (mestre) continuam, com a mesma semantica.
   - `FOREIGN KEY` com `REFERENCES` continua; opcionalmente com `ON DELETE CASCADE`
     para simplificar o apagarFuncionario.
3. Migracoes: sai `PRAGMA user_version`, entra a tabela `schema_migrations`
   (versao, nome, aplicado_em). A funcao `userVersion()` vira `versaoAtual()`.
4. Transacoes: `apagarFuncionario` passa a pegar um client do pool e usar
   `BEGIN/COMMIT/ROLLBACK`.

Contrato de nomes: os metodos publicos do banco mantem os nomes atuais
(`criarMestre`, `buscarUsuarioPorLogin`, `validarSenha`, etc.), agora retornando
promessas. As consultas diretas que os testes fazem hoje via `banco.db.prepare(...)`
passam a usar um helper de consulta exposto pelo banco.

## AppHost (recursos e conexao)

- `AddPostgres("pg").WithDataVolume().AddDatabase("klok")`.
- `AddDockerfile("api", ...)` referenciando o Postgres com `WithReference` e
  `WaitFor`, o que injeta a connection string por variavel de ambiente.
- Segredos e config (`JWT_SECRET`, `SMTP_HOST`, `SMTP_USER`, `SMTP_PASS`,
  client id do Google) por parameters.
- Endpoint HTTP exposto, documentado para o app Expo apontar.

## Dockerfile

- Multi-stage: um estagio instala as dependencias e outro roda.
- Imagem final enxuta de Node 22, usuario nao root, `npm ci --omit=dev`, copiando
  apenas o necessario, com `CMD ["node","server.js"]`.
- `.dockerignore` para nao levar `node_modules`, `.env`, banco local nem docs.

## Testes e CI

- `iniciarApp` (em `test/helpers.js`) sobe um Postgres efemero via testcontainers,
  cria um banco isolado, roda as migracoes e devolve a URL da API.
- `migracao.test.js` adapta para a tabela `schema_migrations`.
- Os acessos diretos do teste ao driver passam a usar o helper de consulta do
  banco.
- CI (`.github/workflows/ci.yml`): job da API roda `npm test` (com Docker
  disponivel no runner) e um job de typecheck fica opcional, ja que o repo e a
  API, sem o app.

## Assuncoes

- Esquema novo, sem migrar os dados do `klok.db` atual. O banco do Klok e de
  desenvolvimento.

## Riscos

- A API fica toda assincrona. Ajuste mecanico, porem amplo, e varios testes
  precisam de `await`.
- Diferencas de semantica entre SQLite e Postgres (ID, booleano, fuso de datas)
  pedem cuidado nos testes existentes.
- testcontainers baixa a imagem do Postgres no primeiro run e no CI, somando
  alguns segundos.

## Decisoes relacionadas (ADRs)

- 0001 registrar decisoes em ADRs
- 0002 Aspire como orquestrador local
- 0003 AppHost em C#
- 0004 API em container desde o dev
- 0005 Postgres em todos os ambientes
- 0006 testcontainers nos testes
- 0007 publicar docker compose, sem nuvem
- 0008 repositorio separado com copia da API
