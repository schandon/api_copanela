# Notas de atualização

## N001 — PRD da API REST Copanela

**Data:** 02/08/2026  
**Status:** Aprovado

### O que mudou
- Criado o PRD em `tasks/prd-apiRestCopanela/prd.md`, seguindo o template `templates/prd-template.md`.
- Documento define a API REST em TypeScript (Express + Prisma + PostgreSQL) com arquitetura em camadas `entity ↔ repository ↔ service ↔ controller ↔ routes`.

### Comportamento / requisitos principais
- Rota base retorna `API COPANELA ON ⚽`; endpoints de domínio versionados em `/api/v1`.
- Autenticação JWT (secret no `.env`), senha com bcrypt; papéis `Jogador`, `Admin`, `Gerente`.
- Com JWT válido: listar, criar, editar e excluir; cadastro e login públicos.
- Usuário só se torna jogador ao informar sexo, lado de quadra e categoria.
- CRUD completo das entidades e pivôs (`Duo_Camp`, `CT_Camp`, `CT_Jogador`).
- `CT.plano`: `free` | `premium`.
- Campeonato com 1..N CTs; inscrição de duplas sem quórum mínimo nesta versão.
- Confrontos/partidas com campeonato, duplas, placar, data/hora, fase e status.
- Envelope de resposta: `data`, `erro`, `mensagem`; CORS via `.env`.

### Fora de escopo (registrado)
- Pagamentos, chat, ranking federativo, upload de mídia e diferenciação fina de permissões por papel (além do acesso autenticado).

### Motivo
- Formalizar requisitos aprovados para guiar Tech Spec e implementação do backend consumido pelo frontend existente.

## N002 — Tech Spec da API REST Copanela

**Data:** 02/08/2026  
**Status:** Aprovado

### O que mudou
- Criada a Tech Spec em `tasks/prd-apiRestCopanela/techspec.md`, seguindo o template `templates/techspec-template.md`.
- Traduz o PRD em decisões de arquitetura e implementação (greenfield em `src/`).

### Decisões técnicas principais
- Organização por **feature** com camadas internas `entity → repository → service → controller → routes`.
- Stack: TypeScript **ESM**, Express 5, Prisma/PostgreSQL, Zod, `jsonwebtoken`, `bcryptjs`, cors, dotenv, helmet, morgan, pino.
- Auth com **access token + refresh token**; papel inicial `Usuario`, promoção para `Jogador` ao criar perfil.
- IDs **UUID**, soft delete (`deletedAt`), listagens com `page`/`limit`.
- Confrontos com enums de fase/status e placar em **JSON estruturado**.
- Envelope `{ data, erro: boolean, mensagem }`.
- Testes: Vitest + Supertest com integração em banco; Playwright adiado (sem UI neste repositório).
- Observabilidade: logs estruturados (morgan/pino); Prometheus/Grafana fora desta versão.

### Motivo
- Orientar a geração de tasks e a implementação do backend com contratos e ordem de construção claros.
