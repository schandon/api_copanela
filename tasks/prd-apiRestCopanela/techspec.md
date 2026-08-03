# Template de Especificação Técnica

## Resumo executivo

Implementar a API REST Copanela do zero em TypeScript (ESM) com Express 5, Prisma e PostgreSQL, organizada por **feature** e camadas internas `entity → repository → service → controller → routes`. Autenticação com **access JWT + refresh token**, senhas com `bcryptjs`, validação com Zod, CORS/Helmet/Morgan/Pino e envelope `{ data, erro, mensagem }`.

`src/` está vazio: bootstrap completo (config, Prisma schema, middlewares, módulos de domínio, testes Vitest+Supertest com banco). Preferir bibliotecas maduras (`jsonwebtoken`, `bcryptjs`, `zod`, `cors`, `dotenv`, `helmet`, `morgan`, `pino`) em vez de soluções customizadas.

## Arquitetura do Sistema

### Visão Geral dos Componentes

**Novos componentes**

| Componente | Responsabilidade |
|---|---|
| `src/App.ts` | Factory Express: middlewares globais, montagem `/api/v1`, error handler |
| `src/Server.ts` | Bootstrap HTTP (`HOST`/`PORT`), graceful shutdown Prisma |
| `src/config/Ambiente.ts` | Validação do `.env` com Zod |
| `src/config/Prisma.ts` | Singleton `PrismaClient` |
| `src/shared/RespostaApi.ts` | Helper `{ data, erro, mensagem }` |
| `src/shared/Erros.ts` | Erros de domínio (`NaoEncontrado`, `Validacao`, `NaoAutorizado`) |
| `src/middlewares/AutenticacaoMiddleware.ts` | Valida Bearer access token |
| `src/middlewares/ValidacaoMiddleware.ts` | Aplica schemas Zod (body/query/params) |
| `src/middlewares/TratamentoErrosMiddleware.ts` | Normaliza exceções → envelope + HTTP |
| `src/modules/saude` | `GET /` → `API COPANELA ON ⚽` |
| `src/modules/auth` | Cadastro, login, refresh |
| `src/modules/usuario` | CRUD usuário (soft delete) |
| `src/modules/jogador` | Perfil jogador (promove papel) |
| `src/modules/categoria` | CRUD categoria |
| `src/modules/dupla` | CRUD dupla |
| `src/modules/centroTreinamento` | CRUD CT + vínculo alunos (`CT_Jogador`) |
| `src/modules/campeonato` | CRUD campeonato + `CT_Camp` + `Duo_Camp` |
| `src/modules/confronto` | CRUD confrontos/placar estruturado |
| `prisma/schema.prisma` | Modelo UUID + enums + `deletedAt` |
| `tests/` | Vitest unitário + integração Supertest/DB |

**Relacionamentos:** Routes → Controller → Service → Repository → Prisma. Entity tipa o domínio sem I/O.

**Fluxo de dados:** HTTP → CORS/Helmet/Morgan → (JWT se protegido) → Zod → Controller → Service (regras) → Repository (Prisma, filtra `deletedAt: null`) → envelope JSON.

## Design de Implementação

### Interface Principais

```ts
// AuthService
interface AuthService {
  cadastrar(entrada: CadastrarUsuarioDto): Promise<{ usuario: UsuarioPublico; accessToken: string; refreshToken: string }>;
  login(entrada: LoginDto): Promise<{ usuario: UsuarioPublico; accessToken: string; refreshToken: string }>;
  renovarToken(refreshToken: string): Promise<{ accessToken: string; refreshToken: string }>;
}

// JogadorService
interface JogadorService {
  criarPerfil(usuarioId: string, entrada: CriarJogadorDto): Promise<Jogador>;
  listar(filtros: Paginacao): Promise<Pagina<Jogador>>;
  obterPorId(id: string): Promise<Jogador>;
  atualizar(id: string, entrada: AtualizarJogadorDto): Promise<Jogador>;
  excluir(id: string): Promise<void>; // soft delete
}

// CampeonatoService
interface CampeonatoService {
  criar(entrada: CriarCampeonatoDto): Promise<Campeonato>;
  vincularCt(campeonatoId: string, ctId: string): Promise<void>;
  inscreverDupla(campeonatoId: string, duplaId: string): Promise<void>;
  listar(filtros: Paginacao): Promise<Pagina<Campeonato>>;
  excluir(id: string): Promise<void>;
}

// ConfrontoService
interface ConfrontoService {
  criar(entrada: CriarConfrontoDto): Promise<Confronto>;
  atualizarPlacar(id: string, placar: PlacarJson): Promise<Confronto>;
  listarPorCampeonato(campeonatoId: string, filtros: Paginacao): Promise<Pagina<Confronto>>;
}
```

### Modelos de Dados

**Enums Prisma**
- `PapelUsuario`: `Usuario` | `Jogador` | `Admin` | `Gerente` (cadastro inicia em `Usuario`; vira `Jogador` ao criar perfil)
- `Sexo`: `Feminino` | `Masculino`
- `LadoQuadra`: `esquerda` | `direita` | `ambos`
- `PlanoCt`: `free` | `premium`
- `FaseConfronto`: `grupos` | `oitavas` | `quartas` | `semifinal` | `final`
- `StatusConfronto`: `agendado` | `em_andamento` | `finalizado` | `wo` | `cancelado`

**Entidades (IDs `uuid`, todos com `createdAt`, `updatedAt`, `deletedAt?`)**
- `Usuario`: nome, email `@unique`, senhaHash, telefone, papel
- `RefreshToken`: id, tokenHash, usuarioId, expiresAt, revokedAt?
- `Jogador`: usuarioId `@unique`, sexo, ladoQuadra, categoriaId
- `Categoria`: nome
- `Dupla`: jogador01Id, jogador02Id, categoriaId, qtdJogos
- `CentroTreinamento`: nome, telefone, instagram, responsavelId, bairro, plano, quantidadeAlunos
- `Campeonato`: nome, quantidadeJogadores, categoriaId, data
- `Confronto`: campeonatoId, dupla01Id, dupla02Id, dataHora, fase, status, placar `Json`
- Pivôs: `DuoCamp`, `CtCamp`, `CtJogador` (também com soft delete)

**Placar JSON (exemplo)**
```json
{
  "sets": [{ "dupla01": 6, "dupla02": 4 }, { "dupla01": 6, "dupla02": 3 }],
  "vencedorDuplaId": "uuid-opcional"
}
```

**Envelope**
- Sucesso: `{ data: T, erro: false, mensagem: string }`
- Erro: `{ data: null, erro: true, mensagem: string }`

**Paginação:** query `page` (default 1), `limit` (default 20, max 100) → `data: { itens, page, limit, total }`.

### Endpoints de API

Públicos:
- `GET /` — texto `API COPANELA ON ⚽`
- `POST /api/v1/auth/cadastro`
- `POST /api/v1/auth/login`
- `POST /api/v1/auth/refresh`

Protegidos (Bearer access JWT) — CRUD + vínculos, todos com paginação nas listagens:
- ` /api/v1/usuarios`
- ` /api/v1/jogadores`
- ` /api/v1/categorias`
- ` /api/v1/duplas`
- ` /api/v1/centros-treinamento` (+ `POST/DELETE .../:id/alunos/:jogadorId`)
- ` /api/v1/campeonatos` (+ `POST/DELETE .../:id/cts/:ctId`, `.../:id/duplas/:duplaId`)
- ` /api/v1/confrontos` (+ filtro `campeonatoId`; `PATCH .../:id/placar`)

Métodos: `GET` lista/detalhe, `POST` cria, `PUT`/`PATCH` atualiza, `DELETE` soft delete.

## Pontos de Integracão

- **PostgreSQL** via `DATABASE_URL` (Docker `infra/postgres`). Falha de conexão: log error + exit no boot; em request: 500 envelope.
- **Frontend** (consumidor): CORS via `ALLOWED_*` do `.env`. Sem outros serviços externos no MVP.
- Auth: access curto (ex. 15m) + refresh longo (ex. 7d) persistido com hash; refresh rotacionado; revoke no logout futuro (opcional).

## Abordagem de Testes

### Testes Unidade

- Services: auth (hash/login/refresh), jogador (promoção de papel), confronto (placar/fase/status), regras de vínculo CT/dupla.
- Mock apenas de repositories/interfaces; sem mock de HTTP.
- Criticos: email duplicado, login inválido, soft delete oculto em listagens, inscrição livre sem quórum.

### Testes de Integração

- Vitest + Supertest + Prisma contra Postgres de teste (`DATABASE_URL` de CI/local).
- Fluxos: cadastro→login→refresh; criar jogador; dupla; CT+aluno; campeonato+CT+dupla; confronto+placar; listagens `page/limit`.
- Setup: migrate/push + limpeza entre suites (truncate ou DB isolado).

### Testes de E2E

- Nesta versão da API: cobertura HTTP via integração Supertest (não há frontend neste repositório).
- Playwright (`./tests`) fica para quando houver UI neste monorepo; desvio justificado da rule de front.

## Sequenciamento de Desenvolvimento

### Ordem de Construção

1. Bootstrap ESM/TS, Ambiente, Prisma schema/migrations, App/Server, envelope, middlewares, saúde.
2. Auth (cadastro/login/refresh) + Usuario.
3. Categoria → Jogador → Dupla.
4. CentroTreinamento + `CtJogador`.
5. Campeonato + `CtCamp` + `DuoCamp`.
6. Confronto (placar JSON, fase, status).
7. Testes unitários/integração + scripts npm (`dev`, `build`, `test`, `prisma:migrate`).

### Dependências Técnicas

- Postgres disponível (`infra/postgres` ou equivalente).
- Node LTS com suporte ESM.
- Variáveis: `SECRET`, `DATABASE_URL`, `PORT`, `HOST`, `DEBUG`, CORS; adicionar `JWT_ACCESS_EXPIRES`, `JWT_REFRESH_EXPIRES`, `BCRYPT_ROUNDS`.

## Monitoramento e Observabilidade

Sem Prometheus/Grafana nesta versão.

- **HTTP:** `morgan` (combined em prod; dev curto).
- **App:** `pino` (info/warn/error); em `DEBUG=true`, level debug.
- Logs: request id opcional, rota, status, duração, `usuarioId` se autenticado; nunca logar senha/token em claro.
- Métricas Prometheus: fora de escopo (adiar).

## Considerações Técnicas

### Decisões Principais

| Decisão | Justificativa | Alternativa rejeitada |
|---|---|---|
| Feature folders + camadas internas | Escala por domínio e respeita entity↔routes | Pastas só por camada global |
| ESM | Pedido explícito; alinhado Node moderno | Manter CommonJS |
| Zod | Inferência TS + middleware simples | class-validator / joi |
| bcryptjs + jsonwebtoken | Maduros, tipados, sem native build pesado | bcrypt nativo; Passport completo |
| Access + refresh | Melhor segurança mobile/web | Só access longo |
| UUID | Estável para sync frontend | Int autoincrement |
| Soft delete | Preserva histórico de campeonatos | Hard delete |
| Placar JSON | Flexível a sets/games sem migrar colunas | String livre / colunas fixas |
| Papel inicial `Usuario` | Usuário ≠ jogador até criar perfil | Já nascer como `Jogador` |
| Vitest + Supertest + DB | Adequado a API pura | Playwright-only neste repo |

### Riscos Conhecidos

| Risco | Mitigação |
|---|---|
| Contrato frontend ainda fluido | Envelope estável; versionar `/api/v1` |
| Refresh token leak | Hash no DB, rotação, expiry, HTTPS em prod |
| Soft delete + unique email | Partial unique index `WHERE deletedAt IS NULL` ou reuso controlado |
| Compose Node legado (`infra/nodejs`) | Não bloquear MVP; ajustar imagem depois |
| N+1 Prisma em listagens | `include`/`select` explícitos nos repositories |
| Papel `Usuario` além do PRD (3 papéis) | Documentar extensão necessária ao fluxo “não jogador” |

### Conformidade com Padrões

- **`.cursor/rules/codigo_padrao.mdc`**: código/identificadores em português; camelCase funções/vars; **PascalCase arquivos**; early returns; métodos curtos; camadas em `/src`.
- **`.cursor/rules/specify-rules.mdc`**: fluxo de aprovação + `notas.md`; testes Playwright em `./tests` — **desvio**: API usa Vitest+Supertest+DB; Playwright quando houver UI neste repo.
- **`.cursor/rules/react.mdc`**: N/A (backend); consumidor futuro segue View→Controller→Model→API.

### Arquivos relevantes e dependentes

- `tasks/prd-apiRestCopanela/prd.md` (fonte de requisitos)
- `package.json` (migrar para `"type": "module"`, scripts e deps)
- `.env` / `.env-example` (estender JWT/bcrypt)
- `infra/postgres/docker-compose.yml` (DB local)
- `infra/nodejs/*` (deploy futuro; não bloqueante)
- Novos: `src/**`, `prisma/schema.prisma`, `tests/**`, `tsconfig.json`, `vitest.config.ts`
