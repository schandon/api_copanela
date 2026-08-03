# Template de Documento de Requisitos de Produto (PRD)

## Visão Geral

A API REST Copanela é o backend responsável por gerenciar o ecossistema de beach tennis da plataforma: usuários, jogadores, centros de treinamento (CT), duplas, categorias, campeonatos, vínculos entre entidades e confrontos com placar. O frontend já existente consumirá essa API para cadastro, autenticação e operação dos fluxos de campeonato.

O problema central é a ausência de um backend estruturado que permita organizar campeonatos com um ou vários CTs, transformar usuários em jogadores mediante dados esportivos, formar duplas, inscrever participantes e acompanhar confrontos. A API entrega essa capacidade de forma versionada, autenticada e padronizada para consumo pelo frontend.

## Objetivo

- Disponibilizar uma API REST em TypeScript com Express, Prisma e PostgreSQL, seguindo o modelo de camadas `entity ↔ repository ↔ service ↔ controller ↔ routes`.
- Permitir autenticação JWT com secret em `.env` e senha armazenada exclusivamente com hash bcrypt.
- Entregar CRUD completo de todas as entidades do domínio e pivôs (`Duo_Camp`, `CT_Camp`, `CT_Jogador`).
- Viabilizar os fluxos prioritários: cadastro/login, perfil de jogador, montagem de dupla, cadastro de CT com vínculo de alunos, criação de campeonato com inscrição de duplas e montagem/listagem de confrontos com placar.
- Critérios de sucesso mensuráveis do MVP:
  - Rota base responde `API COPANELA ON ⚽`.
  - CRUD completo das entidades e pivôs operacionais.
  - Login JWT funcionando.
  - Frontend consome com sucesso criação de usuários, criação de duplas e listagem de partidas/confrontos.
- Objetivos de negócio: centralizar a operação de campeonatos multi-CT e reduzir fricção na gestão de jogadores, duplas e confrontos.

## Histórias de Usuário

**Personas primárias**
- Usuário Padrão: conta criada sem perfil esportivo completo.
- Jogador: usuário que informou lado de quadra e categoria.
- Responsável do CT / Gerente: administra CT e alunos (um jogador pode também ser responsável de CT).
- Organizador do Campeonato / Gerente: cria campeonatos, vincula CTs e gerencia inscrições/confrontos.
- Admin: gestão total do sistema.

**Papéis JWT (autorização):** `Jogador`, `Admin`, `Gerente`.

Com JWT válido, o usuário autenticado PODE **listar, criar, editar e excluir** recursos do domínio (CRUD). Cadastro e login permanecem públicos.

**Histórias**
- Como Usuário Padrão, quero me cadastrar e fazer login para acessar a plataforma com segurança.
- Como Usuário Padrão, quero completar lado de quadra e categoria para me tornar Jogador e participar de duplas/campeonatos.
- Como Jogador, quero formar uma dupla com outro jogador para competir em uma categoria.
- Como Gerente (responsável de CT), quero cadastrar o CT, definir plano (`free` ou `premium`) e vincular alunos para organizar minha base.
- Como Gerente (organizador), quero criar um campeonato vinculado a 1..N CTs e inscrever duplas para disputar a competição.
- Como usuário autenticado, quero montar confrontos (com data/hora, fase, status e placar) e listar partidas para o frontend.
- Como Admin, quero gerenciar categorias e entidades do sistema para manter a integridade do domínio.
- Caso extremo: usuário já jogador também atua como responsável de CT sem criar segunda conta.
- Caso extremo: campeonato sem CT vinculado ou sem duplas inscritas não deve avançar para montagem de confrontos válidos.
- Caso extremo: não há quórum mínimo de duplas por categoria nesta versão (inscrição livre).

## Funcionalidades Principais

### 1. Saúde e versão da API
- O que faz: expõe rota base e versionamento (`/api/v1`).
- Por que é importante: confirma disponibilidade e contrato estável para o frontend.
- Alto nível: rota raiz retorna texto fixo; demais recursos sob prefixo versionado.
- Requisitos funcionais:
  - RF01: A rota base da aplicação DEVE retornar exatamente `API COPANELA ON ⚽`.
  - RF02: Os endpoints de domínio DEVEM ser versionados sob `/api/v1`.
  - RF03: A API DEVE habilitar CORS conforme `ALLOWED_ORIGINS`, `ALLOWED_METHODS`, `ALLOWED_HEADERS` e `ALLOWED_CREDENTIALS` do `.env`.

### 2. Autenticação e autorização
- O que faz: cadastro, login e proteção de rotas por JWT e papéis.
- Por que é importante: segurança do acesso e identificação do usuário nas operações de domínio.
- Alto nível: credenciais validadas; token JWT emitido; CRUD do domínio exige JWT válido.
- Requisitos funcionais:
  - RF04: O sistema DEVE permitir cadastro de usuário com nome, email único, senha e telefone.
  - RF05: A senha DEVE ser persistida apenas com hash bcrypt.
  - RF06: O login DEVE autenticar por email/senha e retornar JWT assinado com `SECRET` do `.env`.
  - RF07: A autorização DEVE reconhecer os papéis `Jogador`, `Admin` e `Gerente`.
  - RF08: Endpoints de domínio autenticados (listar, criar, editar e excluir) DEVEM exigir JWT válido; cadastro e login DEVEM permanecer públicos.

### 3. Evolução Usuário → Jogador
- O que faz: transforma usuário em jogador ao completar dados esportivos.
- Por que é importante: só jogadores entram em duplas e campeonatos.
- Alto nível: criação/atualização de perfil Jogador ligado ao Usuário.
- Requisitos funcionais:
  - RF09: Um usuário SÓ se torna jogador ao informar `sexo`, `lado_quadra` e `fk_categoria`.
  - RF10: `sexo` DEVE aceitar apenas `Feminino` ou `Masculino`.
  - RF11: `lado_quadra` DEVE aceitar apenas `esquerda`, `direita` ou `ambos`.
  - RF12: Um mesmo usuário DEVE poder ser jogador e responsável de CT simultaneamente.

### 4. Gestão de domínio (CRUD completo)
- O que faz: CRUD de Usuário, Jogador, Categoria, Dupla, Centro de Treinamento, Campeonato, Confronto/Partida e pivôs.
- Por que é importante: base operacional completa para o frontend.
- Alto nível: operações criar, listar, obter, atualizar e remover por entidade, com validações de FK e unicidade.
- Requisitos funcionais:
  - RF13: A API DEVE oferecer CRUD completo para Usuário, Jogador, Categoria, Dupla, Centro de Treinamento, Campeonato e Confronto/Partida.
  - RF14: Categoria DEVE possuir ID e nome; Dupla e Campeonato DEVEM referenciar categoria via `fk_categoria`.
  - RF15: Dupla DEVE referenciar `fk_jogador_01`, `fk_jogador_02`, `fk_categoria` e `qtd_jogos`.
  - RF16: A relação Dupla↔Campeonato DEVE ser N:N exclusivamente via pivô `Duo_Camp` (não usar FK única de dupla no campeonato).
  - RF17: A relação Campeonato↔CT DEVE ser N:N via pivô `CT_Camp` (campeonato organizado por 1..N CTs).
  - RF18: A relação CT↔Jogador (alunos) DEVE ser N:N via pivô `CT_Jogador`.
  - RF19: Centro de Treinamento DEVE incluir nome, telefone, Instagram, responsável (`fk_usuario`), bairro, plano (`free` ou `premium`), quantidade de alunos e vínculos de alunos.
  - RF20: Campeonato DEVE incluir nome, quantidade de jogadores, categoria, data e vínculos com CTs e duplas.
  - RF21: Email de usuário DEVE ser único.
  - RF22: `CT.plano` DEVE aceitar apenas os valores `free` e `premium`.

### 5. Fluxos de campeonato e confrontos
- O que faz: criação de campeonato, inscrição de duplas e montagem/listagem de confrontos com placar e metadados de partida.
- Por que é importante: núcleo do valor de negócio e critério de sucesso com o frontend.
- Alto nível: organizador cria campeonato, vincula CTs, inscreve duplas e registra confrontos (data/hora, fase, status, placar).
- Requisitos funcionais:
  - RF23: A API DEVE permitir cadastrar CT e vincular/desvincular alunos.
  - RF24: A API DEVE permitir criar campeonato e vincular um ou mais CTs.
  - RF25: A API DEVE permitir inscrever e listar duplas em um campeonato via `Duo_Camp`.
  - RF26: A API DEVE permitir montar confrontos (partidas) associados a um campeonato, com identificação das duplas envolvidas.
  - RF27: Cada confronto DEVE contemplar, no mínimo: campeonato, duplas envolvidas, placar, data/hora, fase e status.
  - RF28: A API DEVE permitir registrar e atualizar placar, fase, status e data/hora de cada confronto.
  - RF29: A API DEVE permitir listar confrontos/partidas para consumo do frontend.
  - RF30: Nesta versão, NÃO DEVE haver quórum mínimo obrigatório de duplas por categoria (inscrição livre).

### 6. Contrato de resposta e validação
- O que faz: padroniza payload de sucesso/erro e valida entradas.
- Por que é importante: facilita integração estável com o frontend ainda em evolução.
- Alto nível: envelope comum + validação de campos obrigatórios/enums/FKs.
- Requisitos funcionais:
  - RF31: Respostas da API DEVEM seguir o formato com campos `data`, `erro` e `mensagem`.
  - RF32: Entradas inválidas DEVEM retornar erro claro sem persistir dados inconsistentes.
  - RF33: Configurações sensíveis (`SECRET`, `DATABASE_URL`, CORS, `PORT`, `HOST`) DEVEM vir do `.env`.

## Experiência do usuário

Embora o produto seja uma API, a experiência relevante é a do consumidor (frontend) e dos papéis de negócio.

- Personas e necessidades: cadastro simples; promoção a jogador com poucos campos; gestão de CT (plano free/premium) e campeonato sem ambiguidade de vínculos; listagem clara de confrontos com data/hora, fase, status e placar.
- Fluxos principais:
  1) Cadastro → Login → JWT.
  2) Completar perfil de jogador.
  3) Montar dupla.
  4) Cadastrar CT (plano `free` ou `premium`) → vincular alunos.
  5) Criar campeonato → vincular CTs → inscrever duplas (sem quórum mínimo) → montar confrontos → atualizar placar/status → listar partidas.
- UI/UX (contrato de API): mensagens previsíveis (`data`/`erro`/`mensagem`), códigos HTTP coerentes, versionamento explícito e CORS configurável para o frontend local/remoto.
- Acessibilidade/usabilidade da API: erros compreensíveis, validação consistente e documentação implícita pelo contrato versionado para facilitar uso por desenvolvedores do frontend.

## Restrições Técnicas de ALto nível

- Stack obrigatória desta versão: TypeScript, Express, Prisma ORM, PostgreSQL.
- Arquitetura obrigatória em camadas: `entity ↔ repository ↔ service ↔ controller ↔ routes`.
- Autenticação JWT com `SECRET` em `.env`; senhas com bcrypt.
- Com JWT válido, operações de listar/criar/editar/excluir do domínio são permitidas; papéis reconhecidos: `Jogador`, `Admin`, `Gerente`.
- `CT.plano` restrito a `free` | `premium`.
- Confronto/Partida inclui no mínimo: campeonato, duplas, placar, data/hora, fase e status.
- Sem quórum mínimo de duplas por categoria nesta versão.
- String de conexão PostgreSQL via `DATABASE_URL` em `.env`.
- CORS e parâmetros de host/porta via `.env` (conforme `.env-example` existente).
- Contrato de resposta padronizado: `data`, `erro`, `mensagem`.
- Versionamento de API em `/api/v1`.
- Integração com frontend existente (contrato ainda não 100% fechado); a API deve manter estabilidade do envelope de resposta.
- Dados sensíveis: senha nunca em texto puro; tokens e secrets fora do código-fonte.
- Conformidade de domínio alinhada a práticas comuns de campeonatos de beach tennis (duplas por categoria; inscrição de duplas; organização por entidade promotora/CT), sem exigir chancela federativa nesta versão.
- Metas de performance desta versão: suporte adequado a uso local/integração inicial do frontend; SLAs rígidos de TPS ficam para etapas futuras.

Detalhes de implementação serão abordados na Especificação Técnica.

(Nota: Riscos de implementação técnica serão detalhados na Tech Spec.)

## Fora de Escopo

- Pagamentos, taxas de inscrição e gateways financeiros.
- Chat ou mensageria entre usuários.
- Ranking federativo, pontuação oficial CBBT/ITF e chancela de federações.
- Upload de imagens/mídia e gestão de arquivos.
- App mobile nativo dedicado.
- Notificações push/email transacional avançado.
- Geração automática avançada de chaves/algoritmos oficiais de sorteio federado (além da montagem/listagem de confrontos com placar e metadados prevista no MVP).
- Definição final e imutável do contrato do frontend (ainda em evolução); ajustes finos de payload podem ocorrer após alinhamento.
- Quórum mínimo obrigatório de duplas por categoria (fica livre nesta versão; pode ser introduzido depois).
- Diferenciação fina de permissões por papel além do acesso autenticado via JWT (refinamento futuro, se necessário).
