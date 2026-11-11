# Plataforma de Inteligência & Governança de Tráfego Pago

> Proposta de arquitetura para o sistema operacional central da agência.
> Status: 🟡 arquitetura de triangulação decidida; plano de MVP detalhado em
> [`mvp-plano-detalhado.md`](mvp-plano-detalhado.md).

## Visão (revisada)

A plataforma é **banco de dados organizado + painel de visualização** — nada
além disso. Ela guarda todo o contexto do cliente (onboarding, estratégias,
método, histórico, métricas já coletadas), mostra isso num painel rápido, e
cuida de três tarefas recorrentes: **lembretes, relatórios e alertas**. Ela
tem uma IA interna própria só para montar estratégias a partir desses dados —
mas **ela não escreve em Meta Ads nem Google Ads, e não publica nada**.

A escrita — qualquer ação que altere uma conta de anúncio ou publique
conteúdo — é responsabilidade do **Claude Code**, operado com sessão MCP
conectada diretamente à Meta/Google. É um fluxo supervisionado por humano
(o operador abre a sessão, decide, executa), não uma automação headless.

### Triangulação

```
┌───────────────┐        leitura         ┌───────────────────┐
│  Meta Ads /    │ ───────────────────▶  │                    │
│  Google Ads    │                        │    CLAUDE CODE     │
│  (API, só      │ ◀─────────────────────│  (sessão com MCP)  │
│   LEITURA no   │   escrita (futuro,     │                    │
│   MVP)         │   supervisionada)      └─────────┬──────────┘
└───────────────┘                                    │
                                          MCP da Plataforma
                                           (ler contexto/método,
                                            gravar métricas e logs)
                                                      │
                                                      ▼
                                       ┌──────────────────────────┐
                                       │      PLATAFORMA           │
                                       │  Banco de dados + Painel  │
                                       │  + IA interna (estratégia)│
                                       │  + Relatórios/Alertas/    │
                                       │    Lembretes via Telegram │
                                       └──────────────────────────┘
```

**Quem faz o quê:**

| Peça | Papel |
|---|---|
| **Meta/Google API** | No MVP, só leitura (`ads_read`) de métricas de campanha. Escrita fica fora de escopo até estar madura. |
| **Claude Code + MCP** | Ponte entre a Meta/Google e a Plataforma. Lê métricas da Meta/Google, grava na Plataforma via MCP. No futuro, quando a escrita entrar, também é quem executa ajustes de campanha e publicação — direto na conta, numa sessão supervisionada. |
| **Plataforma** | Banco de dados + painel. Guarda onboarding, método, histórico, métricas já coletadas. Tem IA interna própria (via API da Anthropic, não Claude Code) para gerar estratégias, calcular Health Score, montar relatório quinzenal e disparar alertas/lembretes pelo Telegram. **Nunca chama a API da Meta/Google diretamente.** |
| **Telegram** | Canal de notificação — briefing do dia, alertas de anomalia, lembretes de tarefa. Não é um app, é mensagem. |

> **Por que essa divisão importa:** a Plataforma nunca precisa de permissão de
> escrita na Meta (`ads_management`) nem de App Review para publicação
> orgânica — isso elimina o maior gargalo de aprovação externa do projeto.
> Só a sessão interativa do Claude Code, operada por um humano, vai precisar
> dessas permissões, e só quando esse módulo for de fato construído.

> ⚠️ **Relação com o Althos CRM:** ainda pendente de decisão — ver seção de
> decisões pendentes.

---

## 1. Módulos funcionais da Plataforma (o que ela de fato faz)

### Módulo 1 — Painel do Gestor (Central de Anomalias & Saúde Operacional)

- **Health Score** por cliente (verde/amarelo/vermelho), calculado pela IA
  interna sobre as métricas já sincronizadas no banco — não consulta a Meta
  em tempo real.
- **Detecção de anomalias**: ROAS fora da faixa, CPL/CPA atípico, pacing de
  orçamento, com base no snapshot de métricas mais recente.
- **Gargalos operacionais**: etapas travadas (bruto aguardando edição,
  aprovação de cliente pendente, reunião quinzenal sem pauta).
- Qualquer anomalia relevante gera um **alerta via Telegram**.

### Módulo 2 — Visão Panorâmica da Carteira

Cards/linhas por cliente: investimento, conversões, CPA/CPL, ROAS (dados do
último sync) e fase estratégica atual (nomenclatura a alinhar com
`01-metodo/`).

### Módulo 3 — Hub 360° do Cliente

- Onboarding & contexto do negócio.
- Biblioteca de método (espelha `01-metodo/`, não duplica).
- **Approval Hub**: upload de bruto → pipeline de edição → portal de
  aprovação do cliente. A aprovação só muda o status no banco e dispara um
  alerta Telegram pro operador — **a publicação em si é feita depois, pelo
  operador, via Claude Code** (ver seção 4).
- Relatórios e pauta de reunião quinzenal (ver Módulo 5).

### Módulo 4 — Lembretes, Relatórios e Alertas via Telegram

Este é o módulo que materializa "ela vai ter o painel, mas vai me avisar pelo
Telegram":

- **Briefing diário** — resumo curto, enviado automaticamente, do que mudou
  desde ontem (sem precisar abrir o painel).
- **Alertas de anomalia** — disparados pelo Módulo 1, no momento em que a IA
  interna detecta algo fora do esperado no snapshot mais recente.
- **Lembretes operacionais** — reunião quinzenal sem pauta, aprovação de
  cliente pendente há X dias, bruto recebido sem produção iniciada.
- **Relatório quinzenal** — a IA interna cruza os últimos 15 dias e manda um
  resumo (ou o link do PDF) direto pelo Telegram, além de deixá-lo na
  Plataforma.

> O Telegram é só canal de saída. Nenhuma ação é **tomada** a partir dele no
> MVP — é leitura passiva de notificação, não um bot de comando. Ver
> `mvp-plano-detalhado.md` para se isso muda depois (bot que responde a
> comandos, ex. `/status cliente-x`).

### Módulo 5 — IA Interna da Plataforma

Separada do Claude Code. É a camada que:

- Lê o banco de dados da própria Plataforma (nunca a Meta/Google
  diretamente).
- Monta rascunhos de estratégia (próximo ângulo, estrutura de teste) a
  partir do onboarding + método + métricas já sincronizadas.
- Calcula Health Score e detecta anomalias.
- Gera o texto do relatório quinzenal e dos alertas/lembretes.

Implementação técnica: chamadas à API da Anthropic (não ao Claude Code),
rodando em cron/job da própria Plataforma. Detalhe de modelo e padrão de
chamada em `mvp-plano-detalhado.md`.

---

## 2. O que fica fora do MVP (mas mapeado para depois)

- **Escrita em Meta/Google Ads** (pausar anúncio, ajustar orçamento, criar
  campanha) — via Claude Code + MCP, sessão supervisionada por humano. Exige
  `ads_management` em Full Access (App Review + Business Verification).
- **Publicação orgânica** (Feed/Stories/Reels) — mesma lógica: executada via
  Claude Code + MCP quando chegar a hora, não pela Plataforma. Exige
  `instagram_content_publish`/`pages_manage_posts`, também sob review.
- **Geração de criativo** (ElevenLabs, Nano Banana, Runway) — mantido fora de
  escopo do MVP; se entrar, é orquestrado pelo Claude Code, não pela
  Plataforma.

Guardrails de autonomia (teto por alteração, teto diário, log `OC-{VERT}-XXX`
já descritos no `06-tecnologia/README.md`) continuam valendo quando a escrita
for implementada — e ficam mais simples de aplicar porque há sempre um
humano operando a sessão do Claude Code.

---

## Decisões pendentes

- [ ] Reconciliar com o Althos CRM: módulo, substituição ou sistema paralelo?
- [ ] Modelo de dados de onboarding — reaproveitar o que já existe no CRM?
- [ ] Nomenclatura de fases estratégicas — alinhar com `01-metodo/`
- [ ] Hospedagem e stack — avaliar frente à decisão já tomada em
      `06-tecnologia/README.md` (Cloudflare Pages) vs. Vercel+Supabase
      (decidido para este projeto novo — ver `mvp-plano-detalhado.md`)
- [ ] Confirmar se `ads_read` para contas de cliente compartilhadas via
      "Assign Partner" exige Business Verification mesmo em modo leitura
- [ ] Repositório novo, conta Vercel nova e projeto Supabase novo — a
      construir fora deste repositório (`scatto-performance`)
