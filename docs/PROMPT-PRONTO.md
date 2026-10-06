# Prompt pronto (sem nicho definido, com caça de produtos virais)

Cole o bloco abaixo inteiro no Claude Code, dentro deste repositório.

```
Sou afiliada da Shopee Brasil, iniciante, e ainda não tenho nicho definido.
Quero um pipeline automatizado que (1) descubra os produtos com maior chance
de vender e viralizar, (2) gere conteúdo original e (3) poste nas redes.
Antes de escrever código, entre em modo de planejamento, me mostre a
arquitetura e as fases, e espere minha aprovação.

CONTEXTO
- Nicho: ainda não tenho. O sistema deve TESTAR 3 nichos em paralelo por
  14 dias e me recomendar em qual focar com base nos dados:
    A) Achadinhos para casa: cozinha, limpeza e organização (até R$60)
    B) Beleza acessível: skincare, maquiagem e cabelo (até R$50)
    C) Gadgets e acessórios de celular/tecnologia (até R$80)
  Os nichos ficam num arquivo config/nichos.yaml para eu trocar depois
  (ex.: pet, bebê, fitness, acessórios femininos).
- Público: mulheres e homens de 18 a 40 anos que buscam "achadinhos"
  baratos e úteis no TikTok e no Instagram.
- Plataformas: TikTok (prioridade), Instagram Reels, YouTube Shorts.
- Formatos: carrossel de fotos 1080x1920 E vídeo vertical de 15-30s,
  sem precisar aparecer o meu rosto.
- Orçamento: começar perto de R$0 (ferramentas gratuitas) e no máximo
  R$100/mês depois que der resultado.
- Tenho: conta de afiliado Shopee (vou pedir AppID + Secret da Open API).
  Ferramenta de agendamento: Metricool.

RESTRIÇÕES (obrigatórias)
- Nunca repostar vídeos de terceiros nem o vídeo original da Shopee:
  todo conteúdo precisa de roteiro, narração e montagem próprios.
- Credenciais só por variáveis de ambiente (.env no .gitignore, GitHub Secrets).
- Todo passo precisa funcionar em modo --dry-run (gera arquivos e não posta).
- Respeitar os limites das APIs (TikTok: app não auditado só posta privado,
  então use o agendador parceiro ou envio como rascunho).
- Legenda com aviso de link de afiliado / #publi.
- Não prometer resultados de saúde/estética nem usar alegações falsas.

FASES

1. CAÇA DE PRODUTOS (o coração do sistema)
   a) Coleta: cliente da Shopee Affiliate Open API
      (https://open-api.affiliate.shopee.com.br/graphql, autenticação SHA256
      com AppID+Timestamp+Payload+Secret). Use productOfferV2 para buscar
      por palavra-chave, categoria e ordenação por vendas/comissão,
      paginando até ~200 candidatos por nicho por dia.
   b) Sinais de tendência externos (gratuitos, com tolerância a falhas):
      Google Trends Brasil (pytrends) para as palavras-chave do produto,
      e uma lista manual de "termos em alta" em config/tendencias.yaml que
      eu possa atualizar (ex.: o que vejo no TikTok Creative Center).
   c) Análise visual com Claude (visão) da foto principal, nota de 0 a 10 em:
      - Efeito "uau"/novidade (a pessoa nunca viu isso?)
      - Demonstrável em 3 segundos (antes/depois, problema→solução)
      - Resolve uma dor comum do dia a dia
      - Apelo de compra por impulso
   d) Score de potencial (pesos em config/score.yaml):
        30% sinais de venda   (vendas totais e ritmo recente, avaliação ≥4.7)
        25% potencial viral   (nota visual da etapa c)
        20% ganho por venda   (preço x % de comissão)
        15% preço de impulso  (ideal entre R$15 e R$80, desconto alto)
        10% tendência         (Google Trends / termos em alta)
      Filtros eliminatórios: avaliação < 4.5, poucas vendas, loja sem
      reputação, produtos proibidos ou de marca falsificada.
   e) Gere generateShortLink só para os aprovados, salve tudo em SQLite
      e não repita produto em menos de 30 dias.
   f) Relatório diário em output/AAAA-MM-DD/ranking.md com o Top 10 de
      cada nicho e o motivo de cada nota.

2. ROTEIRO: Claude API gera em JSON: gancho (até 8 palavras), 3
   benefícios, CTA "link na bio, produto nº X", legenda e 5 hashtags.
   Crie 3 variações de gancho e escolha a melhor com uma rubrica.
   Use uma biblioteca de modelos de gancho em config/ganchos.yaml
   (ex.: "Comprei isso na Shopee e...", "Ninguém te conta que...",
   "Pare de sofrer com...").

3. MÍDIA:
   a) Carrossel: 4-6 slides com Pillow (capa com gancho grande, fotos do
      produto com texto de benefício, slide final com preço e CTA).
   b) Vídeo: FFmpeg com as fotos em movimento (Ken Burns), narração TTS
      em pt-BR (edge-tts gratuito), legendas queimadas palavra a palavra
      e música sem direitos autorais.
   Três templates visuais em config/templates.yaml, alternados para
   não parecer repetitivo.

4. VITRINE: página estática no GitHub Pages, bonita e rápida no celular,
   com os produtos numerados, foto, preço e link de afiliado, atualizada
   a cada post. É o link da bio.

5. REVISÃO: pasta output/AAAA-MM-DD/ com tudo pronto e um resumo para eu
   aprovar. Flag --auto-aprovar para pular essa etapa no futuro.

6. POSTAGEM: integração com a API do Metricool para agendar no TikTok,
   Instagram e YouTube nos melhores horários; alternativa de enviar ao
   TikTok como rascunho. Camada de postagem plugável.
   Ritmo: 3 posts/dia por plataforma (1 de cada nicho) durante o teste.

7. AGENDAMENTO: workflow do GitHub Actions com cron 2x ao dia que roda o
   pipeline e commita a vitrine atualizada.

8. APRENDIZADO: script semanal que puxa cliques e comissões (API de
   relatórios da Shopee) e views/curtidas/salvamentos, mostra qual nicho,
   gancho e template performam melhor, e ajusta automaticamente os pesos
   do score e a quantidade de posts por nicho. Após 14 dias, gere um
   relatório recomendando o nicho principal.

ENTREGÁVEIS
- Código em Python organizado por módulos, com testes (pytest) e README em
  PT-BR explicado para iniciante, incluindo como pegar as credenciais.
- CLAUDE.md explicando a arquitetura e como rodar.
- Skills do Claude Code:
    /cacar-produtos [nicho]   → roda só a Fase 1 e mostra o ranking
    /gerar-post [produto]     → gera carrossel + vídeo de um produto
- Comece pela Fase 1 (com dados de exemplo enquanto eu não tenho as
  credenciais) + Fase 3a (carrossel), para eu ver o ranking e os slides
  em output/ antes de avançar.
```
