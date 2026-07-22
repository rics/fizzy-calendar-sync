# Specification: Fizzy Calendar Sync

**Author:** Ricardo Silva
**Site pessoal:** https://ricsilva.com
**Repo:** https://github.com/rics/fizzy-calendar-sync
**Status:** revisado — pronto para desenvolvimento (ver `ROADMAP.md`)

> Este documento é a versão 2 da spec original, revisada após checagem contra a
> documentação oficial da API da Fizzy e das políticas do Google/Chrome Web Store.
> Todas as correções e decisões tomadas durante essa revisão estão incorporadas
> abaixo. Trechos marcados com **[DECISÃO]** foram escolhas conscientes tomadas
> na conversa de planejamento; trechos marcados com **[CORREÇÃO]** substituem
> algo que estava errado ou incompleto na v1.

## 1. Overview

Fizzy Calendar Sync é uma extensão de Chrome (Manifest V3) que integra o
gerenciador de tarefas Fizzy diretamente ao Google Calendar. Ela injeta um Side
Panel dentro da aba do Google Calendar, mostra os cards (tarefas) abertos da
Fizzy, e permite agendar horários no Google Calendar clicando ou arrastando os
cards para a agenda.

**Distribuição-alvo:** Chrome Web Store, pública, para qualquer pessoa com uma
conta Google e uma conta Fizzy — **[DECISÃO]** isso implica que o projeto
precisa passar pela verificação de OAuth do Google (scope sensível) e pelo
review completo da Chrome Web Store, não apenas funcionar localmente.

## 2. Core Features & UX Flow

- **Settings & Auth Setup:**
  - Campo para o usuário colar seu **Personal Access Token da Fizzy**
    (persistido via `chrome.storage.local`). **[CORREÇÃO]** Fizzy não tem OAuth
    — o token é gerado manualmente pelo usuário nas configurações da própria
    conta Fizzy, com permissão `read` ou `write`. O onboarding precisa
    explicar isso passo a passo (link direto pra tela de tokens da Fizzy, se
    possível).
  - Google Calendar OAuth via `chrome.identity.getAuthToken` para o scope
    `https://www.googleapis.com/auth/calendar.events`.
- **Side Panel UI:**
  - Busca e lista os cards abertos da Fizzy.
  - Barra de busca/filtro rápido.
  - Botão de atualização (refresh) para sincronizar sob demanda.
  - **[CORREÇÃO]** Precisa lidar com paginação (ver §6-A) se o usuário tiver
    mais cards do que uma página retorna.
  - Rodapé com crédito: "Desenvolvido por Ricardo Silva (ricsilva.com)".
- **Scheduling Workflow:**
  1. Usuário seleciona/clica um card da Fizzy no Side Panel (fica destacado
     como ativo).
  2. Usuário seleciona um horário ou clica em "Agendar" para chamar
     `events.insert` da API do Google Calendar.
  3. Suporta agendamento duplicado (o mesmo card pode ser agendado mais de uma
     vez na agenda) — **[DECISÃO]** mantido por escolha de produto; mitigar
     cliques acidentais com um toast de confirmação visível ("Evento criado às
     14:00"), não com bloqueio.

### 2.1 Casos de erro e borda (não cobertos na v1)

- **Conflito de horário: [DECISÃO]** permitir sobreposição. Ao arrastar um
  card sobre um slot que já tem evento, o evento é criado normalmente, sem
  bloqueio nem confirmação extra — mesmo comportamento padrão do próprio
  Google Calendar.
- **Token expirado:** se o token do Google ou da Fizzy expirar/for revogado no
  meio do uso, o Side Panel deve mostrar um estado claro de "reconectar",
  não uma falha silenciosa.
- **Fizzy fora do ar / rate limit:** a API da Fizzy retorna `429 Too Many
  Requests` em excesso de chamadas. Implementar retry com backoff exponencial
  e, se persistir, mostrar erro amigável (não travar a UI).
- **Sem conexão:** Side Panel deve informar estado offline em vez de lista
  vazia sem explicação.

## 3. Architecture & Tech Stack

- **Manifest Version:** V3
- **Permissions Required:**
  - `storage` (persiste settings/token da Fizzy)
  - `identity` (retrieval do token OAuth do Google)
  - `sidePanel` — **[CORREÇÃO]** o nome correto da permissão é `sidePanel`
    (camelCase), a v1 tinha `sidepanel` em minúsculas.
  - **[DECISÃO]** `activeTab` removida do manifest — não há injeção de
    conteúdo na aba ativa fora do Calendar, e a permissão só adicionaria
    fricção desnecessária no review da Chrome Web Store.
- **oauth2 block no manifest.json** — **[CORREÇÃO]** faltava na v1.
  `chrome.identity.getAuthToken` exige um bloco `oauth2` com `client_id` e
  `scopes` declarado no manifest, não só a permissão `identity`.
  - **Atenção:** o Client ID OAuth do tipo "Chrome Extension" no Google Cloud
    Console é amarrado ao **ID fixo da extensão**. Isso cria uma dependência
    de ordem: ou se sobe a extensão (mesmo unlisted) pra obter o ID antes de
    fechar o client OAuth, ou se fixa o campo `key` no manifest manualmente
    com antecedência. Decidir isso antes de gerar o client ID.
- **Host Permissions:**
  - **[CORREÇÃO]** `https://*.fizzy.do/*` estava errado — a v1 assumia
    subdomínio por conta, mas a Fizzy identifica a conta por **slug no path**
    (`/:account_slug/...`), não por subdomínio. Trocar para:
    - `https://app.fizzy.do/*` (host único e fixo).
  - `https://www.googleapis.com/*`
  - **[DECISÃO]** Fora de escopo: suporte a instâncias self-hosted da Fizzy
    com domínio próprio. `app.fizzy.do` é o único host suportado na v1 — não
    implementar `optional_host_permissions` nem campo de URL base
    configurável.

## 4. Onboarding (novo)

Sequência recomendada na primeira execução:
1. Botão "Conectar com o Google" → dispara `chrome.identity.getAuthToken`.
2. Campo para colar o Personal Access Token da Fizzy, com link/instrução de
   onde gerá-lo (Configurações → API → Personal access tokens → Generate new
   access token, com permissão `read` bastando para o caso de uso de leitura
   de cards).
3. Extensão chama `GET https://app.fizzy.do/my/identity` com o token colado
   para descobrir automaticamente a(s) conta(s) e o `account_slug` — **não**
   pedir esse dado ao usuário (ver §5-A). Se a identidade tiver mais de uma
   conta, deixar o usuário escolher qual usar.
4. Primeira sincronização de cards + mensagem de sucesso.

## 5. API Integrations

### A. Fizzy API

- Referência: https://github.com/basecamp/fizzy/blob/main/docs/api/README.md
- **Autenticação:** header `Authorization: Bearer <FIZZY_PERSONAL_ACCESS_TOKEN>`
  (Personal Access Token gerado manualmente pelo usuário; permissão `read` é
  suficiente para listar cards).
- **Endpoint de listagem:** `GET /:account_slug/cards` — **[CORREÇÃO]** exige
  o slug da conta no path, não é um endpoint global `/cards`.
- **Descoberta do `account_slug` — [RESOLVIDO]** `GET
  https://app.fizzy.do/my/identity` retorna a identidade do usuário e a lista
  de contas às quais ele tem acesso, cada uma já trazendo o campo `slug`
  (ex.: `"slug": "/897362094"`). Chamar esse endpoint logo após o usuário
  colar o token, para obter o(s) `account_slug` automaticamente — nunca pedir
  esse dado digitado manualmente. Se a identidade tiver acesso a mais de uma
  conta, apresentar um seletor simples na tela de onboarding.
- **Filtro de "tarefas abertas":** **[CORREÇÃO]** não existe um valor pronto
  tipo `open`/`unarchived`. O parâmetro mais próximo é `indexed_by`, com
  valores `all` (default), `maybe`, `closed`, `not_now`, `stalled`,
  `postponing_soon`, `golden`. Nenhum desses é exatamente "aberto e não
  arquivado" — **validar em teste manual** se `indexed_by=all` já exclui os
  cards fechados, ou se é preciso filtrar client-side pelo campo `closed`
  (presente no payload de detalhe do card) ou por `column_ids[]` /
  `assignment_status`.
- **Outros parâmetros de filtro úteis:** `board_ids[]`, `tag_ids[]`,
  `assignee_ids[]`, `column_ids[]`, `terms[]` (busca), `sorted_by`.
- **Paginação:** resposta inclui header `Link` com `rel="next"` quando há mais
  páginas — o client precisa seguir esse link, não assumir página única.
- **Rate limiting:** tratar `429 Too Many Requests` com backoff.

### B. Google Calendar API

- Referência: https://developers.google.com/workspace/calendar/api/v3/reference
- **Scope:** `https://www.googleapis.com/auth/calendar.events` (já é o scope
  mínimo correto — só cria/edita eventos, não lê a agenda inteira).
- **Endpoint:** `POST https://www.googleapis.com/calendar/v3/calendars/primary/events`
- **Payload Structure — [CORREÇÃO] incluir `timeZone` explicitamente:**
  ```json
  {
    "summary": "[Fizzy Task Title]",
    "description": "Fizzy Card URL or Task Details",
    "start": { "dateTime": "ISO_8601_STRING", "timeZone": "America/Sao_Paulo" },
    "end": { "dateTime": "ISO_8601_STRING", "timeZone": "America/Sao_Paulo" }
  }
  ```
  Omitir `timeZone` pode causar bugs de horário quando o `dateTime` não traz
  offset UTC explícito. Usar o fuso do navegador do usuário
  (`Intl.DateTimeFormat().resolvedOptions().timeZone`).

## 6. Publicação: Google OAuth + Chrome Web Store [novo]

Como o objetivo é distribuição pública com login de qualquer usuário Google,
os seguintes itens **fazem parte do escopo do projeto**, não são detalhes de
"depois":

- `calendar.events` é classificado como **scope sensível** pelo Google.
  Aplicações públicas que o usam **precisam passar por verificação OAuth**
  antes que qualquer usuário fora de uma lista de até 100 "test users" consiga
  usar sem ver o aviso "o Google não verificou este app".
- Requisitos da verificação:
  - Nome do app, e-mail de suporte, homepage e política de privacidade
    consistentes na tela de consentimento OAuth.
  - **Homepage pública e relevante ao app** (não pode ser uma página pessoal
    genérica que só menciona o projeto de passagem).
  - **Política de privacidade no mesmo domínio da homepage**, linkada na tela
    de consentimento.
  - Possível pedido de vídeo de demonstração do fluxo de autorização.
  - Prazo típico: alguns dias a poucas semanas.
- Chrome Web Store exige separadamente:
  - Formulário de "práticas de dados" (data disclosure) no listing.
  - Justificativa de cada permissão pedida no manifest.
  - Link de política de privacidade no listing (pode reusar o mesmo da
    verificação OAuth).

### 6.1 Decisão de hospedagem [DECISÃO]

- **Homepage oficial do app** (usada no consent screen e no listing da Chrome
  Web Store): página dedicada via **GitHub Pages**, publicada a partir da
  pasta `docs/` do repo `rics/fizzy-calendar-sync`
  (`https://rics.github.io/fizzy-calendar-sync/`).
- **Política de privacidade**: `docs/privacy.html` no mesmo repo — mesmo
  domínio (`github.io`) que a homepage, satisfazendo o requisito do Google.
- **`ricsilva.com`** continua sendo o site pessoal do autor, apenas linkado
  como crédito ("Desenvolvido por Ricardo Silva") — não é usado como homepage
  da verificação, porque é uma página pessoal/portfólio, não uma página sobre
  o app em si (o Google rejeita homepages que não são claramente relevantes ao
  app sob revisão).
- Conteúdo mínimo da privacy policy: quais dados são acessados (Calendar via
  `calendar.events`, cards da Fizzy via PAT), onde ficam armazenados (só
  `chrome.storage.local`, sem backend próprio), que não há venda/publicidade
  com esses dados, como revogar acesso, e e-mail de contato
  (`hello@ricsilva.com`).

## 7. Itens em aberto para validar durante o desenvolvimento

- [ ] Testar `GET /:account_slug/cards` com `indexed_by=all` e confirmar se
      cards fechados aparecem ou não na resposta.
- [ ] Confirmar processo exato de geração do Client ID OAuth (extensão precisa
      de ID fixo antes ou depois do primeiro upload à Chrome Web Store — ver
      recomendação de fixar o campo `key` no manifest com antecedência).

### Decisões já fechadas (não reabrir sem motivo novo)

- Sem suporte a Fizzy self-hosted na v1 — único host: `app.fizzy.do`.
- Conflito de horário: permite sobreposição, sem bloqueio.
- `activeTab` removida do manifest.
- `account_slug` obtido automaticamente via `GET /my/identity`, nunca digitado
  pelo usuário.
