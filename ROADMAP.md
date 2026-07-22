# Roadmap — Fizzy Calendar Sync

Duas etapas sequenciais. A Etapa 2 depende de artefatos gerados na Etapa 1
(ID fixo da extensão, permissões finais), então não deve começar antes da
Etapa 1 estar funcionalmente pronta.

## Etapa 1 — Desenvolvimento do app

Objetivo: extensão funcional, instalável localmente (modo desenvolvedor),
cobrindo o fluxo completo descrito em `spec.md`.

- [ ] Estrutura do `manifest.json` (Manifest V3, permissões corrigidas,
      bloco `oauth2`, host permissions corrigidas).
- [ ] Tela de configuração/onboarding:
  - conectar com Google (`chrome.identity.getAuthToken`)
  - colar Personal Access Token da Fizzy + instrução de onde gerá-lo
  - campo de URL base da Fizzy (default `https://app.fizzy.do`)
- [ ] Side Panel:
  - resolução do `account_slug` via `GET /my/identity` logo após o onboarding
  - listagem de cards via `GET /:account_slug/cards`
  - busca/filtro
  - botão de refresh
  - tratamento de paginação (`Link: rel="next"`)
- [ ] Fluxo de agendamento:
  - seleção de card + horário → `POST .../events` no Google Calendar
  - payload com `timeZone` explícito
  - suporte a agendamento duplicado do mesmo card
  - permitir sobreposição de horário sem bloqueio (decidido)
- [ ] Tratamento de erro:
  - token expirado/revogado (Google ou Fizzy) → estado de "reconectar"
  - rate limit da Fizzy (429) → retry com backoff
  - offline / Fizzy fora do ar → mensagem clara, sem tela vazia sem explicação
- [ ] Testes manuais dos itens em aberto listados em `spec.md` §7
- [ ] Primeiro upload (mesmo que unlisted) à Chrome Web Store para obter o ID
      fixo da extensão, necessário para o Client ID OAuth da Etapa 2

**Critério de conclusão:** extensão instalável via modo desenvolvedor,
fluxo completo (auth → listar cards → agendar) funcionando de ponta a ponta
contra a Fizzy e o Google Calendar reais.

## Etapa 2 — Páginas e publicação

Objetivo: tudo que é exigido para publicação pública na Chrome Web Store com
login de qualquer usuário Google.

- [ ] Criar `docs/index.html` — homepage dedicada ao app (GitHub Pages)
- [ ] Criar `docs/privacy.html` — política de privacidade (mesmo domínio)
- [ ] Ativar GitHub Pages no repo (Settings → Pages → branch `main`, pasta
      `/docs`)
- [ ] Configurar OAuth consent screen no Google Cloud Console:
  - nome do app, e-mail de suporte, homepage, política de privacidade
  - scope `calendar.events` declarado
- [ ] Submeter para verificação de scope sensível do Google (levar em conta
      prazo de dias/semanas — não deixar para a véspera do lançamento)
- [ ] Preencher listing da Chrome Web Store:
  - formulário de práticas/uso de dados
  - justificativa de permissões
  - screenshots, descrição
  - link da política de privacidade
- [ ] Submeter para review da Chrome Web Store
- [ ] Atualizar `docs/index.html` com o link real da Chrome Web Store após
      publicação (o link é placeholder até lá)

**Critério de conclusão:** app publicado e instalável publicamente na Chrome
Web Store, sem aviso de "app não verificado" no login do Google.
