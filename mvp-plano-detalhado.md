# Plano de MVP — Plataforma de Gestão de Tráfego

> Detalhamento de implementação da arquitetura descrita em
> [`plataforma-gestao-trafego.md`](plataforma-gestao-trafego.md). Este plano
> é a base para o repositório novo (fora do `scatto-performance`), com conta
> Vercel e projeto Supabase próprios.

## 1. Escopo do MVP

Dentro:
- Banco de dados + painel (desktop e mobile responsivo, sem app nativo).
- Leitura de métricas Meta/Google via Claude Code + MCP → gravadas na
  Plataforma.
- IA interna (API Anthropic) para Health Score, anomalias, estratégia e
  relatório quinzenal.
- Approval Hub (upload → pipeline → aprovação do cliente) — sem publicação
  automática.
- Notificações via Telegram (briefing, alerta, lembrete, relatório).

Fora (mapeado, não construído agora):
- Escrita em Meta/Google Ads.
- Publicação orgânica.
- Geração de criativo (voz/imagem/vídeo).
- App mobile nativo.

---

## 2. Stack técnica

| Camada | Escolha | Por quê |
|---|---|---|
| Frontend | Next.js (App Router) + Tailwind | Responsivo de fábrica, deploy direto na Vercel |
| Hospedagem | Vercel (conta nova) | Cron jobs nativos, deploy por push |
| Banco de dados | Supabase (projeto novo) — Postgres + Auth + Storage | Um serviço cobre banco, login e storage de mídia bruta/aprovada |
| IA interna | API da Anthropic (Claude API, não Claude Code) | Chamadas via Tool Runner, rodando dentro de uma function/cron da própria Vercel |
| Notificação | Telegram Bot API | Simples, sem app mobile próprio |
| MCP da Plataforma | Servidor MCP (Node/TS), hospedado junto do backend | Exposto para o Claude Code ler/gravar contexto |
| Leitura Meta/Google | Executada pelo Claude Code (sessão do operador), não por código da Plataforma | Elimina a necessidade da Plataforma lidar com OAuth/tokens da Meta |

> A Plataforma **nunca guarda token de acesso da Meta/Google**. Quem
> autentica contra a Meta/Google é a sessão do Claude Code (do operador), que
> grava o resultado já processado na Plataforma via MCP.

---

## 3. Modelo de dados (Supabase / Postgres)

Tabelas principais — nomes sugeridos, ajustar ao revisar com o método já
documentado em `01-metodo/` e `05-operacional/`:

```
clients
  id, name, vertical, status, created_at

client_onboarding
  id, client_id, icp, personas, pain_points, avg_ticket, margin,
  competitors, notes, updated_at

strategies
  id, client_id, phase, method_ref, summary, created_by ('ai'|'human'),
  created_at

metrics_snapshots
  id, client_id, source ('meta'|'google'), period_start, period_end,
  spend, conversions, cpa, cpl, roas, raw_payload (jsonb), synced_at,
  synced_by ('claude_code')

health_scores
  id, client_id, score ('verde'|'amarelo'|'vermelho'), reason, computed_at

alerts
  id, client_id, type ('anomaly'|'reminder'|'report'), message,
  sent_to_telegram (bool), created_at, resolved_at

raw_assets
  id, client_id, file_url, uploaded_by, status, created_at

creative_pipeline
  id, raw_asset_id, status ('bruto'|'em_edicao'|'aguardando_aprovacao'|'aprovado'),
  edited_url, client_comment, approved_at

reports
  id, client_id, period_start, period_end, pdf_url, summary, generated_at

change_log
  id, client_id, actor ('claude_code'|'ai_interna'|'human'), action,
  payload (jsonb), created_at

telegram_links
  id, user_id, chat_id, created_at

users
  id, email, role ('admin'|'operador'), telegram_chat_id
```

`change_log` é a trilha de auditoria — toda escrita feita pelo Claude Code
via MCP passa por aqui, mesmo sendo só leitura de Meta/Google por enquanto
(registra o sync). Quando a escrita em Ads entrar, é o mesmo lugar que vira
o log `OC-{VERT}-XXX` mencionado no README de tecnologia.

---

## 4. Telas do MVP

| Tela | Conteúdo | Acesso |
|---|---|---|
| **Login** | Auth via Supabase (e-mail/senha ou magic link) | Operador/admin |
| **Dashboard (carteira)** | Cards por cliente: spend, ROAS, CPA/CPL, Health Score, fase | Operador/admin |
| **Cliente 360°** | Onboarding, método aplicado, histórico de métricas (gráfico), estratégias geradas pela IA | Operador/admin |
| **Onboarding (form)** | Cadastro/edição do contexto do cliente | Operador/admin |
| **Approval Hub — pipeline** | Lista de brutos por status, upload, atribuição de edição | Operador/admin |
| **Approval Hub — portal do cliente** | Link público (token), visualização do criativo, comentário, botão aprovar | Cliente (sem login) |
| **Relatórios** | Lista de relatórios quinzenais, viewer, exportar PDF | Operador/admin |
| **Alertas/Histórico** | Lista de alertas emitidos e `change_log` (auditoria) | Operador/admin |
| **Configurações** | Vincular Telegram (chat_id), gerenciar usuários | Admin |

Tudo responsivo — a "versão mobile" é a mesma interface, sem build separado.

---

## 5. MCP da Plataforma — tools expostas ao Claude Code

**Leitura (Claude Code consulta a Plataforma)**

| Tool | Função |
|---|---|
| `get_client_context(client_id)` | Onboarding + método + fase atual |
| `get_last_sync(client_id, source)` | Data/hash do último snapshot de métricas gravado |
| `list_pending_approvals(client_id?)` | Criativos aguardando ou recém-aprovados |

**Escrita (Claude Code grava na Plataforma)**

| Tool | Função |
|---|---|
| `save_metrics_snapshot(client_id, source, period_start, period_end, metrics)` | Grava métricas lidas da Meta/Google |
| `log_action(client_id, actor, action, payload)` | Registra qualquer ação no `change_log` |
| `save_strategy_draft(client_id, phase, summary)` | Grava rascunho de estratégia (também usada pela IA interna) |

> Não existe tool de escrita em Meta/Google exposta pela Plataforma — essa
> chamada, quando existir, é feita pelo Claude Code diretamente contra a API
> da Meta/Google (via o MCP da Meta, não o da Plataforma), fora deste
> servidor.

---

## 6. Fluxos operacionais

### 6.1 Ingestão de métricas (manual/periódica, via operador)

```
Operador abre sessão Claude Code
       │
       ▼
Claude Code lê Meta/Google (ads_read) ──▶ processa/normaliza
       │
       ▼
Claude Code chama save_metrics_snapshot() via MCP da Plataforma
       │
       ▼
Plataforma grava em metrics_snapshots + log_action() registra em change_log
```

No MVP, esse fluxo é disparado pelo operador (ex.: "Claude, sincroniza as
contas de hoje"). Automatizar via cron é possível depois, mas exige dar à
automação as mesmas credenciais Meta que hoje só a sessão interativa tem —
decisão a tomar separadamente, não um bloqueio de arquitetura.

### 6.2 IA interna — Health Score, anomalias, alertas

```
Cron da Vercel (ex. 2x/dia)
       │
       ▼
Function chama API da Anthropic (Tool Runner)
  tools: ler metrics_snapshots, strategies, client_onboarding (Supabase)
       │
       ▼
Calcula Health Score + detecta anomalia + (se houver) monta mensagem
       │
       ▼
Grava em health_scores / alerts
       │
       ▼
Se houver alerta: envia ao Telegram (Bot API) usando telegram_links
```

### 6.3 Relatório quinzenal

Mesmo mecanismo do 6.2, disparado por cron quinzenal: lê os últimos 15 dias
de `metrics_snapshots` + `creative_pipeline`, gera resumo + PDF (biblioteca
de PDF server-side), grava em `reports`, envia link via Telegram.

### 6.4 Approval Hub

```
Operador faz upload do bruto ──▶ raw_assets
       │
       ▼
Edição (interna ou terceirizada) ──▶ creative_pipeline.status = em_edicao
       │
       ▼
Pronto para aprovação ──▶ status = aguardando_aprovacao
       │
       ▼
Cliente abre portal público (token) ──▶ aprova ou comenta
       │
       ▼
status = aprovado ──▶ alerta Telegram pro operador
       │
       ▼
Publicação/impulsionamento — FORA da Plataforma, via Claude Code (quando
esse módulo existir)
```

---

## 7. Telegram — o que ele faz no MVP

- Canal de saída apenas (sem comandos no MVP — pode evoluir pra bot
  interativo depois, ex. `/status cliente-x`).
- Vínculo simples: usuário gera um código na tela de Configurações, manda
  pro bot, bot grava `chat_id` em `telegram_links`.
- Tipos de mensagem: briefing diário, alerta de anomalia, lembrete
  operacional, link do relatório quinzenal.

---

## 8. Fases de entrega e estimativa de sessões

| Fase | Entrega | Sessões |
|---|---|---|
| 0 — Fundação | Repo novo, Next.js + Supabase + Vercel + auth + deploy | 1–2 |
| 1 — Dashboard & Cliente 360° | Módulos 1–3 (sem Approval Hub) lendo dados mockados/manuais | 2 |
| 2 — MCP da Plataforma | Servidor MCP com as tools da seção 5, testado com Claude Code | 1–2 |
| 3 — Ingestão de métricas | Fluxo 6.1 ponta a ponta (Claude Code → Meta/Google → Plataforma) | 1–2 |
| 4 — Approval Hub | Upload, pipeline, portal público do cliente | 2 |
| 5 — IA interna | Health Score, anomalias, estratégia (fluxo 6.2) | 2 |
| 6 — Telegram | Bot, vínculo de conta, disparo de alertas/briefing | 1 |
| 7 — Relatório quinzenal | Geração de PDF + cron + envio (fluxo 6.3) | 1–2 |
| 8 — Polimento | Auth multiusuário, permissões, testes com dados reais | 1–2 |

**Total: ~12–17 sessões.** Sem dependência de App Review da Meta — o MVP
inteiro roda em modo leitura (`ads_read`), e mesmo esse tier pode começar
sendo alimentado manualmente pelo operador enquanto qualquer aprovação
pendente (se houver) tramita. Isso reduz bastante o tempo de calendário em
relação à versão anterior do plano, que dependia de escrita em Ads desde o
início.

---

## 9. Requisitos externos (checklist)

- [ ] Repositório novo no GitHub (fora do `scatto-performance`)
- [ ] Conta Vercel nova + projeto
- [ ] Projeto Supabase novo
- [ ] Bot do Telegram (BotFather) + token
- [ ] Chave de API da Anthropic (para a IA interna)
- [ ] Acesso Meta/Google em modo leitura — pode ser a própria sessão do
      Claude Code do operador, sem credencial própria da Plataforma
- [ ] Domínio (opcional no MVP — subdomínio da Vercel serve)
