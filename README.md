# PS42 Luanda — System Blueprint

> Documento de arquitectura completo para o sistema de gestão da PlayStation da **42 Luanda**.  
> Define todos os módulos, entidades de dados, fluxos de utilizador e decisões técnicas.

---

## 📋 Visão Geral

| Métrica | Valor |
|---|---|
| Módulos | 21 (10 cadete · 8 staff · 3 públicos) |
| Tabelas DB | 12 |
| Endpoints API | 42+ |
| Fases de Dev | 5 |
| Versão | Blueprint v1.0 |
| Data | Agosto 2025 |

---

## 📑 Índice

1. [Módulos do Sistema](#-módulos-do-sistema)
2. [Esquema da Base de Dados](#-esquema-da-base-de-dados)
3. [Integração API 42 Intra](#-integração-api-42-intra)
4. [Fluxos de Utilizador](#-fluxos-de-utilizador)
5. [Motor de Elegibilidade](#-motor-de-elegibilidade)
6. [Stack Tecnológica](#-stack-tecnológica)
7. [Roadmap de Desenvolvimento](#-roadmap-de-desenvolvimento)

---

## 🧩 Módulos do Sistema

### 🌐 Público

| Mod | Nome | Funcionalidades |
|---|---|---|
| MOD · 00 | **Landing Page** | Status em tempo real da consola, login via 42 OAuth, regulamento resumido |

### 👤 Cadete

| Mod | Nome | Funcionalidades |
|---|---|---|
| MOD · 01 | **Dashboard Pessoal** | Elegibilidade em tempo real, próxima sessão, stats pessoais, conquistas |
| MOD · 02 | **Requisição de Sessão** | Formulário com validação automática de elegibilidade via API 42, horários, companheiros |
| MOD · 03 | **Fila em Tempo Real** | Posição na fila via WebSocket, tempo estimado, notificações push |
| MOD · 04 | **Catálogo de Jogos** | Jogos aprovados, categorias, popularidade, sugestões de novos jogos |
| MOD · 05 | **Rankings & Leaderboard** | Tabelas por jogo, semanais/mensais/all-time, filtro por coligação |
| MOD · 06 | **Conquistas & Badges** | Sistema de gamificação com badges, marcos e conquistas da API 42 |
| MOD · 07 | **Torneios** | Brackets automáticos, inscrições, resultados, calendário de eventos 42 |
| MOD · 08 | **Perfil do Cadete** | Dados integrados da API 42, histórico, nível académico, coligação, conquistas |
| MOD · 09 | **Histórico de Sessões** | Registo completo de sessões, duração, jogos, avaliações, exportação |
| MOD · 10 | **Denúncias** | Submissão anónima ou identificada, categorias, acompanhamento de estado |

### 🔴 Staff

| Mod | Nome | Funcionalidades |
|---|---|---|
| MOD · 11 | **Dashboard Operacional** | Controlo em tempo real, sessão activa, fila, alertas, métricas do dia |
| MOD · 12 | **Gestão de Requisições** | Aprovar/rejeitar pedidos, gerir fila, forçar fim de sessão, extensão de tempo |
| MOD · 13 | **Gestão de Cadetes** | Elegibilidade, histórico disciplinar, bloqueio/desbloqueio manual de acesso |
| MOD · 14 | **Gestão de Jogos** | Aprovar/reprovar jogos, gerir catálogo, responder a sugestões |
| MOD · 15 | **Incidentes & Danos** | Registo de danos ao equipamento, upload de fotos, responsáveis, resolução |
| MOD · 16 | **Painel Disciplinar** | Admoestações, suspensões, indemnizações, proposta ao Conselho Pedagógico |
| MOD · 17 | **Analytics & Relatórios** | Gráficos de uso, horários de pico, exportação PDF |
| MOD · 18 | **Configurações do Sistema** | Horários, limites de sessão, critérios de elegibilidade, bloqueios automáticos |

---

## 🗄️ Esquema da Base de Dados

> **PostgreSQL** via Supabase com **Prisma ORM**. 12 tabelas principais.  
> Dados da API 42 são sincronizados e cacheados localmente.

### Tabelas Principais

<details>
<summary><strong>👤 users</strong></summary>

| Campo | Tipo | Chave |
|---|---|---|
| id | uuid | PK |
| intra_id | int | IDX |
| login | varchar | |
| display_name | varchar | |
| email | varchar | |
| avatar_url | text | |
| role | enum(cadete, staff, admin) | |
| coalition_id | int | FK |
| is_eligible | boolean | |
| is_blocked | boolean | |
| intra_level | float | |
| last_sync_at | timestamp | |
| created_at | timestamp | |

</details>

<details>
<summary><strong>📋 sessions</strong></summary>

| Campo | Tipo | Chave |
|---|---|---|
| id | uuid | PK |
| user_id | uuid → users | FK |
| game_id | uuid → games | FK |
| status | enum(pending, active, done, cancelled) | |
| requested_at | timestamp | |
| started_at | timestamp | |
| ended_at | timestamp | |
| duration_min | int | |
| extended | boolean | |
| ended_by | enum(timer, staff, cadete) | |
| notes | text | |

</details>

<details>
<summary><strong>🎮 games</strong></summary>

| Campo | Tipo | Chave |
|---|---|---|
| id | uuid | PK |
| title | varchar | |
| cover_url | text | |
| category | varchar | |
| max_players | int | |
| is_approved | boolean | |
| approved_by | uuid → users | FK |
| format | enum(physical, digital) | |
| pegi_rating | int | |
| total_sessions | int | |
| suggested_by | uuid → users | FK |

</details>

<details>
<summary><strong>⚖️ disciplinary_records</strong></summary>

| Campo | Tipo | Chave |
|---|---|---|
| id | uuid | PK |
| user_id | uuid → users | FK |
| type | enum(warning, suspension, fine, expulsion) | |
| reason | text | |
| article_violated | varchar | |
| issued_by | uuid → users | FK |
| valid_until | timestamp | |
| is_active | boolean | |
| council_approved | boolean | |
| created_at | timestamp | |

</details>

<details>
<summary><strong>🚨 reports</strong></summary>

| Campo | Tipo | Chave |
|---|---|---|
| id | uuid | PK |
| reporter_id | uuid → users (null=anon) | FK |
| reported_user_id | uuid → users | FK |
| category | enum(conduct, damage, rules, other) | |
| description | text | |
| evidence_urls | text[] | |
| status | enum(open, reviewing, resolved, dismissed) | |
| resolved_by | uuid → users | FK |
| resolution_notes | text | |
| created_at | timestamp | |

</details>

<details>
<summary><strong>🏆 game_results · ⚔️ tournaments · 🎖️ achievements</strong></summary>

**game_results** — Resultados de partidas (session_id, game_id, winner_id, participants, score_data jsonb, match_type, verified)

**tournaments** — Torneios (name, game_id, created_by, status, max_participants, bracket_data jsonb, winner_id, starts_at, ends_at)

**achievements** — Conquistas (slug, name, description, icon, category, condition_type, condition_value, rarity)

</details>

<details>
<summary><strong>⚠️ incidents · ⏰ system_config · 📝 audit_logs · 🔔 notifications</strong></summary>

**incidents** — Incidentes (session_id, responsible_id, type, description, photo_urls, repair_cost, status)

**system_config** — Configurações chave-valor (key IDX, value jsonb, description, updated_by, updated_at)

**audit_logs** — Log de auditoria completo (actor_id, action, entity_type, entity_id, before_data jsonb, after_data jsonb, ip_address)

**notifications** — Notificações (user_id, type, title, body, action_url, is_read, created_at)

</details>

---

## 🔌 Integração API 42 Intra

> Base URL: `https://api.intra.42.fr/v2` — Autenticação via **OAuth2**

| Endpoint | Utilização |
|---|---|
| `GET /oauth/token` | Troca do código OAuth2 por access_token (NextAuth.js) |
| `GET /me` | Dados completos do utilizador autenticado (login, email, level, campus) |
| `GET /me/cursus_users` | Progresso no cursus 42 (nível, órbita, skills) |
| `GET /me/projects_users` | Projectos do cadete — verifica atrasos (Art. 3f) |
| `GET /me/scale_teams` | Histórico de avaliações — média > 3/mês (Art. 3d) |
| `GET /me/coalitions` | Coligação do cadete (usado nos rankings) |
| `GET /me/achievements` | Conquistas académicas da Intra |
| `GET /campus/:id/users` | Lista todos os cadetes do campus 42 Luanda |
| `GET /campus/:id/exams` | Exames agendados → bloqueio automático da PS (Art. 4e) |
| `GET /campus/:id/events` | Eventos obrigatórios → bloqueio automático (Art. 4e) |
| `GET /users/:login/locations` | Logs de presença física no campus |
| `GET /users/:id/patroned` | Relações de mentoria (contexto de perfil) |

### Notas de Integração

- **Rate Limiting**: Cache Redis de 15 min para dados de elegibilidade e 1h para perfil estático.
- **Sincronização**: Cron job a cada hora + sincronização imediata no login.
- **Campus ID**: 42 Luanda tem um campus ID específico — confirmar com o staff.
- **Ritmo (Art. 3c)**: Métrica `level ≤ 22` — confirmar cálculo exacto com o staff pedagógico.

---

## 🔄 Fluxos de Utilizador

### 🔐 Login e Verificação de Elegibilidade
```
Cadete acede → Login 42 OAuth → Fetch dados API 42 → Verificar critérios Art. 3
    ├── ✅ Elegível      → Dashboard completo
    └── ❌ Não elegível → Motivo apresentado + Dashboard limitado
```

### 📋 Requisição de Sessão
```
Acede a /requisição → Verifica elegibilidade + horário + exames
    → Preenche formulário (jogo, horário)
    → Entra na fila (tempo real via WebSocket)
    → Levanta a consola na recepção com ID
    → Sessão activa (timer 1h)
```

### 🚨 Denúncia
```
Detecta infracção → Acede a /denúncias → Preenche (categoria, descrição, provas)
    → Staff recebe alerta em tempo real
    → Staff analisa + decide acção
    → Resolução documentada
```

### ⚖️ Processo Disciplinar
```
Infracção detectada → Staff regista processo disciplinar
    → Proposta ao Conselho Pedagógico
    → Conselho aprova medida
    → Sanção aplicada (suspensão / admoestação)
    → Cadete notificado + bloqueio automático
```

---

## ✅ Motor de Elegibilidade

O sistema verifica automaticamente **8 critérios** definidos no **Art. 3.º** do regulamento:

| # | Critério | Tipo | Verificação |
|---|---|---|---|
| A | Ser Cadete da 42 Luanda | Obrigatório | `API: /me → campus[].id === LUANDA_CAMPUS_ID` |
| B | 2ª Órbita do Holy Graph | Obrigatório | `API: /me/cursus_users → projects validados ≥ 2ª órbita` |
| C | Ritmo ≤ 22 | Métrica | `API: /me/cursus_users → level ≤ 22` |
| D | Média de Avaliações > 3/mês | Métrica Mensal | `API: /me/scale_teams → avg(final_mark) > 3 last 30d` |
| E | Sem Histórico Disciplinar Grave | Obrigatório | `DB: disciplinary_records WHERE is_active=true AND type IN (suspension, expulsion)` |
| F | Situação Académica Regular | Condicional | `API: /me/projects_users → sem projectos em falha crítica` |
| G | Comportamento Adequado | Comportamental | `DB: reports WHERE reported_user_id AND status = resolved (guilty)` |
| H | Sem Sanção Disciplinar em Vigor | Obrigatório | `DB: disciplinary_records WHERE is_active=true AND valid_until > NOW()` |

### Bloqueios Automáticos por Horário e Eventos

| Regra | Detalhe |
|---|---|
| **Horário de funcionamento** | Segunda a Sexta: **08h00 às 17h00** — fora deste horário a consola é bloqueada |
| **Exames (API 42)** | Bloqueio total durante exames agendados |
| **Eventos obrigatórios (API 42)** | Bloqueio total durante eventos de participação obrigatória |

---

## 🛠️ Stack Tecnológica

| Categoria | Tecnologia | Razão |
|---|---|---|
| Framework | **Next.js 14** | App Router, Server Components, API Routes num só projecto |
| Linguagem | **TypeScript** | Tipagem forte — erros em desenvolvimento, não produção |
| Styling | **Tailwind CSS** | Utility-first, rápido e consistente |
| Componentes | **shadcn/ui** | Acessíveis, personalizáveis, sem dependência de biblioteca |
| Autenticação | **NextAuth.js** | Provider OAuth2 nativo para 42 Intra |
| Base de Dados | **PostgreSQL** | Robusto, relacional, suporta jsonb |
| Hosting DB | **Supabase** | PostgreSQL gerido, free tier generoso, real-time nativo |
| ORM | **Prisma** | Schema como código, migrações automáticas, type-safe queries |
| Real-time | **Pusher** | WebSockets geridos para fila e notificações push |
| Cache | **Upstash Redis** | Cache de elegibilidade, rate limiting, sessões temporárias |
| Email | **Resend** | Email transaccional com templates React |
| Upload | **Cloudinary** | Upload de fotos de incidentes com transformações automáticas |
| Gráficos | **Recharts** | Gráficos compostos e responsivos para analytics |
| Deploy | **Vercel** | Integração nativa Next.js, preview deploys automáticos |
| Testes | **Vitest + Playwright** | Unit tests (Vitest) + E2E tests (Playwright) |

---

## 🗺️ Roadmap de Desenvolvimento

### Fase 01 — Fundação
- [ ] Setup Next.js 14 + TypeScript + Tailwind
- [ ] Estrutura de pastas (App Router)
- [ ] Configurar NextAuth.js + 42 OAuth
- [ ] Login funcional com conta 42 Intra
- [ ] Prisma schema completo (12 tabelas)
- [ ] Setup Supabase + migrações
- [ ] Design system base (cores, tipografia)
- [ ] Layout (cadete vs staff) + middleware
- [ ] Landing page pública
- [ ] Deploy inicial no Vercel

### Fase 02 — Core · Requisições e Elegibilidade
- [ ] Motor de elegibilidade (8 critérios)
- [ ] Integração API 42 Intra completa
- [ ] Cache Redis para elegibilidade
- [ ] Dashboard do cadete (Mod.01)
- [ ] Formulário de requisição (Mod.02)
- [ ] Fila em tempo real com Pusher (Mod.03)
- [ ] Dashboard operacional staff (Mod.11)
- [ ] Gestão de requisições staff (Mod.12)
- [ ] Timer de sessão (1h + extensão)
- [ ] Bloqueio automático por horário/exames

### Fase 03 — Gaming Hub
- [ ] Catálogo de jogos aprovados (Mod.04)
- [ ] Sistema de rankings e leaderboard (Mod.05)
- [ ] Registo de resultados de partidas
- [ ] Sistema de conquistas e badges (Mod.06)
- [ ] Módulo de torneios com brackets (Mod.07)
- [ ] Perfil completo do cadete (Mod.08)
- [ ] Histórico de sessões (Mod.09)
- [ ] Gestão de jogos pelo staff (Mod.14)

### Fase 04 — Gestão Disciplinar e Segurança
- [ ] Sistema de denúncias (Mod.10)
- [ ] Gestão de cadetes pelo staff (Mod.13)
- [ ] Registo de incidentes e danos (Mod.15)
- [ ] Painel disciplinar completo (Mod.16)
- [ ] Audit logs de todas as acções
- [ ] Sistema de notificações (email + push)
- [ ] Configurações do sistema (Mod.18)

### Fase 05 — Analytics, PWA e Produção
- [ ] Dashboard analytics staff (Mod.17)
- [ ] Exportação de relatórios PDF
- [ ] PWA (Progressive Web App)
- [ ] Notificações push (Web Push API)
- [ ] Testes unitários (Vitest)
- [ ] Testes E2E (Playwright)
- [ ] Optimizações de performance
- [ ] Documentação técnica completa
- [ ] Deploy em produção + monitorização

---

## 📂 Estrutura de Ficheiros Recomendada

```
ps42-luanda/
├── app/
│   ├── (public)/           # Landing page
│   ├── (cadete)/           # Módulos cadete (auth required)
│   │   ├── dashboard/
│   │   ├── requisicao/
│   │   ├── fila/
│   │   ├── jogos/
│   │   ├── rankings/
│   │   ├── conquistas/
│   │   ├── torneios/
│   │   ├── perfil/
│   │   ├── historico/
│   │   └── denuncias/
│   └── (staff)/            # Módulos staff (role required)
│       ├── dashboard/
│       ├── requisicoes/
│       ├── cadetes/
│       ├── jogos/
│       ├── incidentes/
│       ├── disciplinar/
│       ├── analytics/
│       └── configuracoes/
├── components/
│   ├── ui/                 # shadcn/ui components
│   └── shared/             # Componentes partilhados
├── lib/
│   ├── api42/              # Integração API 42 Intra
│   ├── eligibility/        # Motor de elegibilidade
│   ├── prisma/             # Cliente Prisma
│   └── redis/              # Cliente Upstash Redis
├── prisma/
│   └── schema.prisma       # 12 tabelas
└── middleware.ts            # Auth + role routing
```

---

## 👥 Papéis e Permissões

| Papel | Acesso |
|---|---|
| `public` | Landing page, status consola |
| `cadete` | Módulos 01–10 (requisições, perfil, gaming) |
| `staff` | Todos os módulos + gestão operacional |
| `admin` | Configurações do sistema + todas as permissões |

---

*42 Luanda · Agosto 2025 · Blueprint v1.0*
