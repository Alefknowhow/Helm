# Helm

Plataforma de Inteligência & Governança de Tráfego Pago da Scatto Performance.

Banco de dados organizado + painel de visualização + IA interna para
estratégia, Health Score, relatórios quinzenais e alertas/lembretes via
Telegram. Leitura de métricas e qualquer escrita futura em Meta Ads/Google
Ads acontecem via Claude Code com MCP, numa sessão supervisionada — o Helm
nunca guarda credencial da Meta/Google.

## Documentação

- [`arquitetura.md`](arquitetura.md) — visão geral, triangulação
  Meta/Google ↔ Claude Code ↔ Helm, módulos funcionais, decisões pendentes.
- [`mvp-plano-detalhado.md`](mvp-plano-detalhado.md) — stack técnica, modelo
  de dados, telas, tools MCP, fluxos operacionais e fases de entrega do MVP.

## Status

🔲 Repositório criado, documentação de arquitetura e plano de MVP
transferidos do `scatto-performance`. Implementação ainda não iniciada.

Stack planejada: Next.js + Vercel, Supabase (banco/auth/storage), Telegram
Bot API, API da Anthropic (IA interna), servidor MCP próprio.
