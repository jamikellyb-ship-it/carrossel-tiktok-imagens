# Automação de conteúdo para afiliada Shopee: estratégia e prompt

Pesquisa feita em outubro de 2026.

## 1. O que existe hoje

| Abordagem | Ferramentas | Prós | Contras |
|---|---|---|---|
| Bots de WhatsApp/Telegram com "achadinhos" | Achadinho Pro, Divulgador Inteligente, IA Divulgadora | Prontos, geram link de afiliado sozinhos | Mensalidade; não fazem vídeo original para TikTok/Reels |
| Repostar o vídeo do produto que a Shopee disponibiliza | Bots acima, scripts de download | Zero esforço | **Arriscado**: o TikTok detecta vídeo duplicado (principal causa de shadowban em 2026) e a Shopee vem desativando contas por repost e direitos autorais |
| Gerar vídeo com IA a partir do produto | PipClip, CapCut, Higgsfield, Remotion/FFmpeg + TTS | Conteúdo original, escala bem | Precisa montar o pipeline |
| Carrossel de fotos (TikTok Photo Mode) | Script com Pillow/Canva | Barato, rápido, o TikTok está empurrando esse formato | Menos retenção que vídeo em alguns nichos |

### Regras que mudam a estratégia

- **Shopee Video:** a comissão caiu de 3% para **1,5%** em 01/07/2026 (redes sociais continuam em 3%). A comissão só vale com o produto marcado no botão flutuante do app, e **não há API pública para postar no Shopee Video**. Desde abril de 2026 a Shopee pune conteúdo irrelevante. Ali a postagem continua manual ou semimanual.
- **TikTok:** pela Content Posting API, um app **não auditado só consegue postar em modo privado**. Mesmo auditado, o limite é de mais ou menos 15 posts por dia por criador. Os caminhos possíveis são (a) um agendador que já é parceiro oficial (Metricool, Blotato, Postiz etc.) ou (b) enviar como **rascunho** e publicar com um toque no celular.
- **TikTok sem link clicável:** o link vai para uma página "vitrine" no link da bio, com os produtos numerados ("produto 12 na bio").
- **Instagram Reels** (Graph API, conta Business) e **YouTube Shorts** (Data API) aceitam postagem automática pública com menos burocracia.

## 2. Arquitetura recomendada

```
[cron diário: GitHub Actions ou rotina do Claude Code]
   │
   ├─ 1. Curadoria  → Shopee Affiliate Open API (GraphQL)
   │                  productOfferV2: produtos com boa comissão, vendas e avaliação
   │                  generateShortLink: link de afiliado
   ├─ 2. Roteiro    → Claude API: gancho de 3s, 3 benefícios, CTA, legenda, hashtags
   ├─ 3. Mídia      → imagens do produto → carrossel (Pillow) ou vídeo 9:16
   │                  (FFmpeg/Remotion + narração TTS + legenda queimada + música)
   ├─ 4. Vitrine    → página estática (GitHub Pages) com produtos numerados e links
   ├─ 5. Revisão    → fila "aprovar/rejeitar" (nas primeiras semanas)
   ├─ 6. Postagem   → Metricool/Blotato (TikTok, IG, YT) ou rascunho no TikTok
   └─ 7. Métricas   → relatório de cliques/comissões → realimenta a curadoria
```

Princípios:

1. **Conteúdo original** (roteiro, narração e montagem próprios), nunca repost puro.
2. **Humano no loop** no começo. Depois que a qualidade estiver boa, liga o modo 100% automático.
3. **Segredos fora do código** (AppID/Secret da Shopee, tokens) em variáveis de ambiente ou GitHub Secrets.
4. **Modo dry-run**: gera tudo sem postar, para testar.
5. **Volume com qualidade**: 2 a 4 posts por dia por conta, em horários variados.

## 3. Como pedir ao Claude Code

O que funciona melhor:

- **Um projeto por fases**, não "faça tudo". Peça um plano antes (modo Plan) e aprove fase por fase.
- **Dê o contexto de negócio** (nicho, público, plataformas, orçamento) e as **restrições** acima.
- **Peça entregáveis verificáveis**: testes, dry-run, um exemplo gerado em `output/`.
- **Peça um `CLAUDE.md`** para que as próximas sessões já saibam como o projeto funciona.
- **Peça uma skill** (`.claude/skills/gerar-post/SKILL.md`) para você rodar `/gerar-post` sob demanda, e um **agendamento** para rodar sozinho.

### Prompt mestre (copie e ajuste os campos entre colchetes)

```
Sou afiliada da Shopee Brasil e quero um pipeline automatizado que gere
conteúdo original de produtos e poste nas redes. Antes de escrever código,
entre em modo de planejamento, me mostre a arquitetura e as fases, e espere
minha aprovação.

CONTEXTO
- Nicho: [ex.: casa e cozinha / beleza / utilidades até R$50]
- Público: [ex.: mulheres 20-40 anos que buscam "achadinhos"]
- Plataformas: TikTok (prioridade), Instagram Reels, YouTube Shorts
- Formatos: carrossel de fotos 1080x1920 E vídeo vertical de 15-30s
- Orçamento de ferramentas: [ex.: até R$100/mês]
- Tenho: conta de afiliado Shopee com acesso à Open API (AppID + Secret),
  [Metricool / Blotato / nenhuma ferramenta de agendamento]

RESTRIÇÕES (obrigatórias)
- Nunca repostar vídeos de terceiros nem o vídeo original da Shopee:
  todo conteúdo precisa de roteiro, narração e montagem próprios.
- Credenciais só por variáveis de ambiente (.env no .gitignore, GitHub Secrets).
- Todo passo precisa funcionar em modo --dry-run (gera arquivos e não posta).
- Respeitar os limites das APIs (TikTok: app não auditado só posta privado,
  então use um agendador parceiro ou envio como rascunho).
- Legenda com aviso de link de afiliado / #publi quando aplicável.

FASES
1. Curadoria: cliente da Shopee Affiliate Open API
   (https://open-api.affiliate.shopee.com.br/graphql, autenticação SHA256 com
   AppID+Timestamp+Payload+Secret). Use productOfferV2 para buscar produtos
   por palavra-chave/categoria, ordenar por comissão x vendas x avaliação,
   e generateShortLink para gerar o link. Guarde histórico em SQLite para
   não repetir produto em menos de [30] dias.
2. Roteiro: Claude API gera em JSON: gancho (até 8 palavras), 3 benefícios,
   CTA "link na bio, produto nº X", legenda, 5 hashtags. Crie 3 variações
   de gancho e escolha a melhor com uma rubrica.
3. Mídia:
   a) Carrossel: 4-6 slides com Pillow (capa com gancho grande, fotos do
      produto com texto de benefício, slide final com preço e CTA).
   b) Vídeo: FFmpeg (ou Remotion) com as fotos em movimento (Ken Burns),
      narração TTS em pt-BR (edge-tts gratuito, opção ElevenLabs),
      legendas queimadas palavra a palavra e música sem direitos autorais.
   Templates visuais configuráveis em um arquivo YAML (cores, fontes).
4. Vitrine: página estática no GitHub Pages com os produtos numerados,
   foto, preço e link de afiliado, atualizada a cada post. É o link da bio.
5. Revisão: gere uma pasta output/AAAA-MM-DD/ e um resumo (ou e-mail/
   mensagem) para eu aprovar. Flag --auto-aprovar para pular essa etapa.
6. Postagem: integração com [Metricool API / Blotato API] para agendar no
   TikTok, Instagram e YouTube nos melhores horários; alternativa de enviar
   ao TikTok como rascunho. Deixe a camada de postagem plugável.
7. Agendamento: workflow do GitHub Actions com cron [2x ao dia] que roda
   o pipeline e commita a vitrine atualizada.
8. Métricas: script semanal que puxa cliques/comissões (API de relatórios
   da Shopee) e views, e ajusta os pesos da curadoria.

ENTREGÁVEIS
- Código em Python organizado por módulos, com testes (pytest) e README em PT-BR.
- CLAUDE.md explicando a arquitetura e como rodar.
- Skill do Claude Code em .claude/skills/gerar-post/SKILL.md para eu rodar
  /gerar-post [palavra-chave] sob demanda.
- Comece pela Fase 1 + Fase 3a (carrossel) com dados de exemplo, para eu
  ver um resultado em output/ antes de avançar.
```

### Prompts de acompanhamento úteis

- `Rode o pipeline em --dry-run com a palavra-chave "[x]" e me mostre os slides gerados.`
- `Os ganchos estão genéricos. Reescreva o prompt de roteiro usando estes 10 ganchos que viralizaram como exemplo: [...]`
- `Crie 3 templates visuais diferentes e alterne entre eles para evitar conteúdo repetitivo.`
- `Adicione um relatório semanal com os 10 produtos que mais converteram.`

## 4. Ordem sugerida para começar (sem gastar quase nada)

1. Pedir acesso à **Open API** no painel de afiliados da Shopee (AppID + Secret).
2. Rodar o prompt mestre, só Fase 1 + carrossel, e postar **manualmente** por 1 semana para validar nicho e ganchos.
3. Ligar a vitrine e o vídeo com TTS.
4. Ligar a postagem automática via agendador e o cron.
5. Para o Shopee Video (1,5%), reaproveitar os mesmos vídeos gerados e subir pelo app, marcando o produto no botão flutuante.

## Fontes

- [Atualização na comissão de Shopee Video (Central de Ajuda Shopee)](https://help.shopee.com.br/portal/10/article/223989-Atualiza%C3%A7%C3%A3o-na-Comiss%C3%A3o-de-Shopee-Video)
- [Shopee Video afiliado: como postar e ganhar comissão](https://achadinho.pro/blog/shopee-video-para-afiliados-como-postar-e-ganhar-comissao)
- [Melhores ferramentas para afiliados Shopee em 2026](https://achadinhopro.com.br/blog/melhores-ferramentas-afiliados-shopee-2026)
- [Shopee Affiliate Open API Explorer](https://open-api.affiliate.shopee.com.br/explorer/v2)
- [Documentação não oficial da Shopee Affiliate API](https://github.com/bcat95/shopee-aff)
- [TikTok Content Posting API (oficial)](https://developers.tiktok.com/doc/content-posting-api-reference-photo-post)
- [TikTok API Guide 2026: audit e limites (PostZen)](https://www.postzen.dev/blog/tiktok-api)
- [TikTok Automation 2026: o que é permitido](https://openhosst.com/blog/tiktok-automation)
- [Shadowban no TikTok em 2026](https://360uniquizer.com/en/news/shadowban-tiktok-instagram-2026)
- [Vídeo para Shopee com IA (PipClip)](https://pipclip.ai/blog/video-produto-shopee-ia/)
