# ConteinerKlok Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Subir a API do Klok (em container) e um Postgres (em container) orquestrados por um AppHost do Aspire, com a camada de banco portada de SQLite para Postgres e um docker compose gerado por `aspire publish`.

**Architecture:** O repositorio e autocontido e contem `api/` (a klok-api copiada e portada para Postgres), `aspire/ConteinerKlok.AppHost/` (AppHost em C# que descreve a API como imagem e o Postgres) e `.github/workflows/ci.yml`. A API fala com o Postgres por um `Pool` do `pg`; a connection string chega por variavel de ambiente injetada pelo Aspire. Os testes sobem um Postgres efemero via testcontainers.

**Tech Stack:** Node.js 22, Express 4, `pg` (node-postgres), Postgres 17, testcontainers (`@testcontainers/postgresql`), .NET 10 com Aspire (AppHost em C#), Docker e docker compose.

**Spec:** `docs/superpowers/specs/2026-10-13-conteinerklok-design.md`

## Global Constraints

- Node 22 em tudo. O Node do sistema e 12; invocar com `mise exec node@22 --` (ex: `mise exec node@22 -- npm test`).
- Postgres 17 (`postgres:17-alpine`) tanto nos testes quanto no container.
- Comportamento da API preservado: as mesmas rotas, os mesmos codigos HTTP e as mesmas mensagens de erro.
- **Identificadores camelCase vao entre aspas duplas em todo SQL.** O Postgres dobra identificadores sem aspas para minusculas, entao `"senhaHash"`, `"funcionarioId"`, `"dataHora"`, `"senhaVersao"`, `"tokenHash"`, `"expiraEm"`, `"usuarioId"` e aliases como `AS "funcionarioNome"` precisam de aspas. Sem isso, `SELECT *` devolve `senhahash` e o contrato quebra.
- Nomes de tabela em minusculo sem aspas: `funcionarios`, `usuarios`, `pontos`, `recuperacao_senha`, `schema_migrations`.
- Placeholders do `pg` sao `$1`, `$2`, ...; nao usar `?`.
- Comportamento observavel preservado: as rotas, mensagens e codigos de status nao mudam; as 79 asseracoes atuais continuam valendo contra Postgres.
- Texto gerado (docs, mensagens de commit, issues) sem hifen nem travessao.
- `JWT_SECRET` continua obrigatorio para subir a API.

## Review Focus

- Identificador camelCase sem aspas: o Postgres dobra para minusculo e o JS passa a ler `undefined` (ex: `usuario.senhaHash`). Um teste de esquema deve provar que a coluna volta com o nome camelCase.
- Unicidade do mestre em corrida: no Postgres a violacao e o codigo `23505`, nao os codigos do SQLite. O tratamento precisa mapear para 409.
- `expiraEm` como `timestamptz`: o `pg` devolve `Date`; a logica `new Date(registro.expiraEm)` e a comparacao de expiracao devem seguir corretas.
- `usado` como `boolean`: `!registro.usado` e `registro.usado` devem continuar funcionando.
- `await` esquecido: funcao do banco chamada sem `await` devolve `Promise` e o controller responde `{}`. A suite atual (assercoes 2xx) pega isso.

---

### Task 1: Estrutura do repositorio e copia da API (issue #3)

**Files:**
- Create: `api/` (copia de `~/klok/klok-api`, sem `node_modules`, `.env`, `*.db`, `server.log`)
- Modify: `.gitignore` (garantir `node_modules/`, `.env`, `*.db`)
- Keep: `api/package.json` e `api/package-lock.json` como estao (ainda com `node:sqlite`)

**Interfaces:**
- Consumes: nada.
- Produces: a pasta `api/` com a API rodando como hoje; base para as proximas tasks.

- [ ] **Step 1: Copiar a API para o repo novo**

```bash
cd /home/northoncardoso/ConteinerKlok
mkdir -p api
(cd /home/northoncardoso/klok/klok-api && \
  find . -type f -not -path './node_modules/*' -not -name '.env' \
    -not -name '*.db' -not -name '*.db.bak*' -not -name 'server.log' \
    -exec cp --parents {} /home/northoncardoso/ConteinerKlok/api/ \;)
ls api
```
Esperado: os arquivos da API (`app.js`, `db.js`, `server.js`, `controladores/`, `rotas/`, `test/`, `package.json`, `package-lock.json`, `.env.example`).

- [ ] **Step 2: Conferir o `.gitignore` da raiz**

Garantir que contem `node_modules/`, `.env`, `*.db` e `server.log`. O template Node ja cobre `node_modules`; acrescentar o resto se faltar.

- [ ] **Step 3: Rodar a suite de origem para ter a linha de base**

```bash
cd /home/northoncardoso/ConteinerKlok/api
mise exec node@22 -- npm ci
mise exec node@22 -- npm test
```
Esperado: 79 testes passando (ainda em SQLite com `node:sqlite`).

- [ ] **Step 4: Commit**

```bash
git add api .gitignore
git commit -m "chore: copia a API do Klok para api/ (linha de base em SQLite)"
```

---

### Task 2: Portar a camada de banco para Postgres e o harness de teste (issues #4 e #7)

**Files:**
- Modify: `api/package.json` (deps: `pg`; devDeps: `@testcontainers/postgresql`)
- Create: `api/test/postgres.js`
- Modify: `api/db.js` (reescrita para Postgres, async)
- Modify: `api/server.js`
- Modify: `api/middlewares.js`
- Modify: `api/controladores/auth.js`
- Modify: `api/controladores/funcionarios.js`
- Modify: `api/controladores/pontos.js`
- Modify: `api/test/helpers.js`
- Modify: `api/test/migracao.test.js`
- Modify: `api/test/seguranca.test.js`
- Modify: `api/test/recuperacao.test.js`
- Modify: `api/test/auth.test.js` (trocar so os acessos `s.banco.db.prepare`)
- Create: `docs/adr/0009-identificadores-camelcase-entre-aspas.md`

**Interfaces:**
- Consumes: `api/` da Task 1.
- Produces:
  - `criarBanco({ connectionString })` assincrono, devolvendo `Promise` com os metodos publicos atuais (`criarMestre`, `buscarUsuarioPorLogin`, `buscarUsuarioPorId`, `validarSenha`, `baterPonto`, `listarTodosPontos`, etc.), agora assincronos, mais `versaoAtual(): Promise<number>`, `consultar(sql, params): Promise<Resultado>` e `fechar(): Promise<void>`.
  - `iniciarPostgres(): Promise<{ connectionString: string }>` em `api/test/postgres.js`, com um container singleton por processo e um banco novo por chamada.
  - `iniciarApp()` de `api/test/helpers.js` montando a API sobre esse Postgres.

- [ ] **Step 1: Escrever o helper de Postgres para os testes (vai falhar)**

Criar `api/test/postgres.js` com `iniciarPostgres()`:
```js
import { PostgreSqlContainer } from '@testcontainers/postgresql';
import { Client } from 'pg';

let containerPromise;
function container() {
    containerPromise ??= new PostgreSqlContainer('postgres:17-alpine').start();
    return containerPromise;
}

export async function iniciarPostgres() {
    const c = await container;
    const nomeBanco = `klok_${Date.now()}_${Math.floor(Math.random() * 1e6)}`;
    const admin = new Client({ connectionString: c.getConnectionUri() });
    await admin.connect();
    await admin.query(`CREATE DATABASE "${nomeBanco}"`);
    await admin.end();
    const url = new URL(c.getConnectionUri());
    connectionString = `postgres://${url.username}:${url.password}@${url.hostname}:${url.port}/${nomeBanco}`;
    return { connectionString };
}
```
Registrar o singleton do container para encerrar no fim do processo (opcional: `process.on('beforeExit', ...)`).

- [ ] **Step 2: Adicionar as dependencias**

```bash
cd /home/northoncardoso/ConteinerKlok/api
mise exec node@22 -- npm install pg
mise exec node@22 -- npm install --save-dev @testcontainers/postgresql
```

- [ ] **Step 3: Escrever o teste de esquema (camelCase entre aspas) e faze-lo falhar**

Em `api/test/migracao.test.js`, novo teste no topo:

```js
test('o esquema preserva o nome camelCase das colunas', async () => {
    const { connectionString } = await iniciarPostgres();
    const banco = await criarBanco({ connectionString });
    await banco.criarMestre('mestre', 'senha-mestre-teste', 'Mestre', 'mestre@klok.app', '11999999999');
    const linhas = await banco.consultar('SELECT * FROM usuarios WHERE usuario = $1', ['mestre']);
    assert.ok(linhas.rows[0].senhaHash, 'a coluna senhaHash precisa voltar com esse nome');
    await banco.fechar();
});
```
Rodar `mise exec node@22 -- node --test test/migracao.test.js`. Esperado: falhar (o `criarBanco` ainda e SQLite sincrono).

- [ ] **Step 4: Reescrever `api/db.js` para Postgres**

Estrutura:
- `import pg from 'pg'; const { Pool } = pg;`
- `export const MIGRACOES = [...]` com o mesmo array atual, porem em DDL Postgres. A versao 1 vira:

```sql
CREATE TABLE IF NOT EXISTS funcionarios (
    id integer GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    nome text NOT NULL,
    numero text DEFAULT '',
    email text DEFAULT ''
);
CREATE TABLE IF NOT EXISTS usuarios (
    id integer GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    usuario text UNIQUE,
    "senhaHash" text,
    tipo text NOT NULL DEFAULT 'funcionario',
    "funcionarioId" integer REFERENCES funcionarios(id),
    googleId text UNIQUE
);
CREATE TABLE IF NOT EXISTS pontos (
    id integer GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    "funcionarioId" integer NOT NULL REFERENCES funcionarios(id),
    "dataHora" timestamptz NOT NULL,
    tipo text NOT NULL DEFAULT 'batida'
);
```
  A versao 2 (`nome`, `numero`, `email` em usuarios) vira `ALTER TABLE usuarios ADD COLUMN ...`.
  A versao 3 (`recuperacao_senha`) troca `datetime('now')` por `now()` e `usado integer` por `usado boolean NOT NULL DEFAULT false`.
  A versao 4 (`senhaVersao`) vira `ALTER TABLE usuarios ADD COLUMN "senhaVersao" integer NOT NULL DEFAULT 0`.
  A versao 5 (indice do mestre) vira `CREATE UNIQUE INDEX IF NOT EXISTS idx_usuarios_mestre ON usuarios (tipo) WHERE tipo = 'mestre';`
  As migracoes v1 e v2 que usavam `usuario`/`senhaHash` devem citar os nomes entre aspas onde for camelCase (`senhaHash`, `funcionarioId`); `usuario`, `nome`, `numero`, `email`, `tipo`, `id` ficam sem aspas por serem minusculos.

- `export async function criarBanco({ connectionString })`:
  - cria `new Pool({ connectionString })`;
  - `aplicarMigracoes()`: cria `schema_migrations (versao integer primary key, nome text not null, aplicadoEm timestamptz not null default now())` se nao existir; le `SELECT versao FROM schema_migrations`; para cada migracao nao aplicada roda em transacao (`BEGIN`; `migracao.sql`; `INSERT INTO schema_migrations (versao, nome) VALUES ($1,$2)`; `COMMIT`, com `ROLLBACK` em erro);
  - define as funcoes publicas, todas `async`, usando `pool.query` com placeholders `$n`. Regras de conversao:
    - `lastInsertRowid` vira `INSERT ... RETURNING id` e le `res.rows[0].id`.
    - `.changes > 0` vira `res.rowCount > 0`.
    - `apagarFuncionario` pega `const client = await pool.connect()`, `BEGIN`, os tres `DELETE`, `COMMIT`/`ROLLBACK`, `client.release()`, retorna `rowCount > 0`.
    - `validarSenha` continua usando `createHash`/`scryptSync` (sincrono) e faz o `UPDATE` do hash legado com `await pool.query`.
    - `paraTexto`, `criarHashSenha` e `verificarSenha` seguem iguais (sincronos).
  - expoe `versaoAtual()` (`SELECT COALESCE(MAX(versao),0) AS v FROM schema_migrations`), `consultar(sql, params) => pool.query(sql, params)` e `fechar()` (`pool.end()`).

- [ ] **Step 5: Ajustar `api/server.js`**

Trocar `criarBanco()` por uma inicializacao assincrona lendo `process.env.DATABASE_URL ?? process.env['ConnectionStrings__klok']`, num IIFE com `top-level await` (o arquivo ja e ESM):

```js
const connectionString = process.env.DATABASE_URL ?? process.env.ConnectionStrings__klok;
const banco = await criarBanco({ connectionString });
const app = criarApp({ banco });
```
Falhar com mensagem clara se a connection string nao existir.

- [ ] **Step 5b: Ajustar `api/middlewares.js`**

Em `criarAutenticar`, trocar `const usuario = banco.buscarUsuarioPorId(...)` por `const usuario = await banco.buscarUsuarioPorId(...)`. A funcao `autenticar` ja e `async`.

- [ ] **Step 6: Ajustar `api/controladores/funcionarios.js` e `api/controladores/pontos.js`**

Trocar todas as chamadas `banco.x(...)` por `await banco.x(...)` e marcar `listar`, `criar`, `atualizar`, `apagar`, `bater`, `meus`, `todos` como `async`. As rotas ja usam `asyncHandler`, entao erros de Promise sao capturados.

- [ ] **Step 7: Ajustar `api/controladores/auth.js`**

- Marcar como `async` os handlers hoje sincronos: `statusMestre`, `registrar`, `eu`, `atualizarDados`, `alterarSenha`, `paginaRedefinicao`, `redefinirSenha` (os que ja sao async continuam).
- `await` em toda chamada `banco.x(...)`.
- `dadosDeUsuario(usuario)` vira `async` e faz `await banco.buscarFuncionario(...)`; os chamadores fazem `await dadosDeUsuario(...)`.
- `jaExisteMestre(erro)`: trocar os codigos do SQLite por deteccao de Postgres. Aceitar `erro.code === '23505'` e, para o mestre, `erro.constraint === 'idx_usuarios_mestre'`; manter o match por nome na mensagem como fallback.
- `esqueciSenha`: `await enviarEmail(...)` ja existe; `await banco.buscarUsuarioPorEmail`, `await banco.criarRecuperacao`.
- Comparacoes de expiracao seguem com `new Date(registro.expiraEm)`, agora recebendo `Date`.

- [ ] **Step 8: Reescrever `api/test/helpers.js`**

```js
import { criarBanco } from '../db.js';
import { criarApp } from '../app.js';
import { iniciarPostgres } from './postgres.js';

export async function iniciarApp(opcoes = {}) {
    const { connectionString } = await iniciarPostgres();
    const banco = await criarBanco({ connectionString });
    const app = criarApp({ banco, clienteGoogle, enviarEmail: opcoes.enviarEmail, limiteAuth: opcoes.limiteAuth });
    const servidor = app.listen(0);
    await new Promise((r) => servidor.once('listening', r));
    ...
    return { banco, baseUrl, fechar: async () => { servidor.closeAllConnections(); await new Promise((r)=>servidor.close(r)); await banco.fechar(); } };
}
```

- [ ] **Step 9: Adaptar os testes que tocam o banco direto**

- `api/test/seguranca.test.js`: substituir `s.banco.db.prepare('SELECT senhaHash FROM usuarios WHERE usuario = ?').get(x)` por `(await s.banco.consultar('SELECT "senhaHash" FROM usuarios WHERE usuario = $1', [x])).rows[0]`. No insercao de hash legado, usar `await s.banco.consultar('INSERT INTO usuarios (usuario, "senhaHash", tipo) VALUES ($1,$2,$3)', ['legado', hashLegado, 'funcionario'])`. Nos dois testes que fazem `criarBanco(':memory:')`, trocar por `iniciarPostgres` + `criarBanco({ connectionString })` e `await banco.fechar()`.
- `api/test/auth.test.js`: a leitura `s.banco.db.prepare(...)` vira `await s.banco.consultar(...)`. As demais assercoes ficam como estao.
- `api/test/recuperacao.test.js`: `const usuario = await s.banco.buscarUsuarioPorLogin('nina');` e `await s.banco.criarRecuperacao(...)`.
- `api/test/migracao.test.js`: manter o teste novo do Step 3. Reescrever "banco legado migra sem preservar" assim: criar um banco Postgres, aplicar so a `MIGRACOES[0].sql`, inserir `Sobrevivente` e o usuario `antigo`, entao chamar `criarBanco({ connectionString })` e afirmar `versaoAtual() === MIGRACOES.length`, que `listarFuncionarios()` contem `Sobrevivente` e que `senhaVersao` do `antigo` e 0. Manter o teste de ordem das versoes e o de versao de senha, com `await`.

- [ ] **Step 10: Escrever o ADR 0009**

`docs/adr/0009-identificadores-camelcase-entre-aspas.md`: contexto (Postgres dobra identificadores), decisao (manter colunas camelCase entre aspas para preservar o contrato JS), consequencias (SQL com aspas, sem camada de mapeamento).

- [ ] **Step 11: Rodar a suite inteira**

```bash
cd /home/northoncardoso/ConteinerKlok/api
mise exec node@22 -- npm test
```
Esperado: 79 testes passando (mais o teste novo de esquema). Se um teste falhar por coluna `undefined`, e identificador camelCase sem aspas.

- [ ] **Step 12: Commit**

```bash
git add api docs/adr/0009-*.md
git commit -m "feat: porta a camada de banco para Postgres e os testes para testcontainers"
```

---

### Task 3: Dockerfile e .dockerignore da API (issue #5)

**Files:**
- Create: `api/Dockerfile`
- Create: `api/.dockerignore`

**Interfaces:**
- Consumes: `api/` da Task 2 (sobe com `node server.js` e le `DATABASE_URL`).
- Produces: uma imagem que, com `DATABASE_URL` apontando para um Postgres, serve a API na porta `3000`.

- [ ] **Step 1: Escrever o `.dockerignore`**

Conteudo: `node_modules`, `.env`, `*.db`, `*.log`, `test/`, `docs/`.

- [ ] **Step 2: Escrever o `api/Dockerfile`**

```dockerfile
FROM node:22-alpine AS deps
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci --omit=dev

FROM node:22-alpine AS runtime
ENV NODE_ENV=production
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .
USER node
EXPOSE 3000
CMD ["node", "server.js"]
```

- [ ] **Step 3: Subir um Postgres avulso e testar a imagem**

```bash
docker network create ck-teste || true
docker run -d --name pg-teste --network ck-teste -e POSTGRES_USER=test -e POSTGRES_PASSWORD=test -e POSTGRES_DB=klok postgres:17-alpine
cd /home/northoncardoso/ConteinerKlok/api
docker build -t conteinerklok-api:test .
docker run --rm --network ck-teste -e DATABASE_URL="postgres://test:test@pg-teste:5432/klok" -e JWT_SECRET=teste -p 3000:3000 conteinerklok-api:test &
sleep 3
curl -s http://localhost:3000/api/auth/mestre
```
Esperado: `{"cadastrado":false}`. Depois `docker rm -f pg-teste` e `docker network rm ck-teste`.

- [ ] **Step 4: Commit**

```bash
git add api/Dockerfile api/.dockerignore
git commit -m "feat: adiciona Dockerfile multi-stage e .dockerignore da API"
```

---

### Task 4: AppHost em C# com a API e o Postgres (issue #6)

**Prerequisito:** instalar o Aspire CLI (`curl -sSL https://aspire.dev/install.sh | bash`). Conferir com `aspire --version`.

**Files:**
- Create: `aspire/ConteinerKlok.AppHost/ConteinerKlok.AppHost.csproj`
- Create: `aspire/ConteinerKlok.AppHost/AppHost.cs`
- Create: `aspire/ConteinerKlok.AppHost/appsettings.json` (opcional)
- Create: `aspire/ConteinerKlok.sln` (ou `.slnx` via template)

**Interfaces:**
- Consumes: `api/Dockerfile` da Task 3.
- Produces: `aspire start` subindo dois recursos: `pg` (Postgres com volume) e `api` (imagem do Dockerfile), com a connection string injetada.

- [ ] **Step 1: Criar o AppHost**

```bash
cd /home/northoncardoso/ConteinerKlok
aspire add
# escolher o template de AppHost em C#; gerar em aspire/ConteinerKlok.AppHost
```
Ou usar `aspire new aspire-apphost`, ajustando o caminho. O projeto precisa de `Aspire.AppHost.Sdk` e das referencias `Aspire.Hosting.PostgreSQL` e `Aspire.Hosting.Docker` (para o publish).

- [ ] **Step 2: Escrever o `AppHost.cs`**

```csharp
var builder = DistributedApplication.CreateBuilder(args);

var postgres = builder.AddPostgres("pg")
    .WithDataVolume()
    .AddDatabase("klok");

var jwtSecret = builder.AddParameter("jwt-secret", secret: true);

var api = builder.AddDockerfile("api", "../../api")
    .WithHttpEndpoint(targetPort: 3000)
    .WithEnvironment("JWT_SECRET", jwtSecret)
    .WithReference(postgres)
    .WaitFor(postgres);

builder.Build().Run();
```
Se necessario, ler a connection string no `server.js` tambem de `ConnectionStrings__klok` (ja previsto na Task 2).

- [ ] **Step 3: Subir e verificar**

```bash
cd /home/northoncardoso/ConteinerKlok/aspire/ConteinerKlok.AppHost
aspire start
aspire describe
```
Esperado: recursos `pg` e `api` em estado `Running`. Em outro terminal, chamar o endpoint da API mostrado por `aspire describe` em `/api/auth/mestre` e receber `{"cadastrado":false}`. Encerrar com `aspire stop`.

- [ ] **Step 4: Commit**

```bash
git add aspire
git commit -m "feat: adiciona AppHost em C# com a API em container e o Postgres"
```

---

### Task 5: CI com GitHub Actions (issue #7)

**Files:**
- Create: `.github/workflows/ci.yml`

**Interfaces:**
- Consumes: `api/` da Task 2 e o AppHost da Task 4.
- Produces: um workflow que roda a suite no PR e no push para `main`.

- [ ] **Step 1: Escrever o workflow**

```yaml
name: ci
on:
  push:
    branches: [main]
  pull_request:
jobs:
  api:
    runs-on: ubuntu-latest
    defaults:
      run:
        working-directory: api
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '22'
          cache: npm
          cache-dependency-path: api/package-lock.json
      - run: npm ci
      - run: npm test
  apphost:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-dotnet@v4
        with:
          dotnet-version: '10.0.x'
      - run: dotnet build aspire/ConteinerKlok.AppHost/ConteinerKlok.AppHost.csproj
```
O runner ubuntu tem Docker, entao o testcontainers funciona.

- [ ] **Step 2: Rodar a suite no CI**

Abrir um PR de teste (ou empurrar a branch) e conferir o job `api` verde no GitHub Actions.

- [ ] **Step 3: Commit**

```bash
git add .github/workflows/ci.yml
git commit -m "ci: roda a suite da API e compila o AppHost"
```

---

### Task 6: Documentacao e publish para docker compose (issue #8)

**Files:**
- Modify: `README.md`
- Create: `docs/arquitetura.md` (diagrama em texto)
- Create (gerado): artefatos de `aspire publish` (compose)

**Interfaces:**
- Consumes: tudo acima.
- Produces: README com passos reais e um docker compose executavel.

- [ ] **Step 1: Atualizar o README**

Secoes: o que e, arquitetura (com diagrama em texto: `aspire` -> `api` (container) -> `pg` (container)), como rodar (`aspire start`, `npm test` em `api/`, `aspire publish`) e variaveis de ambiente (`JWT_SECRET`, `DATABASE_URL`, SMTP).

- [ ] **Step 2: Gerar e validar o compose**

```bash
cd /home/northoncardoso/ConteinerKlok/aspire/ConteinerKlok.AppHost
aspire publish -o ../../compose
cd /home/northoncardoso/ConteinerKlok/compose
docker compose up -d
curl -s http://localhost:3000/api/auth/mestre
docker compose down
```
Esperado: `{"cadastrado":false}`.

- [ ] **Step 3: Commit**

```bash
git add README.md docs/arquitetura.md compose
git commit -m "docs: instrucoes de uso e artefato de publish em docker compose"
```

---

## Self-Review

**1. Cobertura da spec:** arquitetura (Task 4), estrutura do repositorio (Task 1), camada de banco (Task 2), Dockerfile (Task 3), testes e CI (Tasks 2 e 5), publish (Task 6), riscos (tratados na Task 2 e no Review Focus). Coberto.

**2. Passos:** cada passo tem uma acao verificavel. Os passos de rewrite dao as assinaturas, as regras de conversao e o SQL exato das migracoes, sem transcrever os corpos.

**3. Consistencia de tipos:** `criarBanco({ connectionString })`, `versaoAtual`, `consultar`, `fechar` aparecem iguais na Task 2 e nas tarefas seguintes. `iniciarPostgres` e usado so em testes.

**4. Review Focus:** cada linha tem um teste dono (esquema camelCase na Task 2 Step 3; 23505 no tratamento do mestre; expiracao e `usado` nos testes de recuperacao; `await` coberto pela suite).

**5. Proporcao:** o plano decide as interfaces e o SQL exato (o que o implementador nao pode adivinhar) e deixa os corpos para ele.
