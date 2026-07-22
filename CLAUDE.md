# CLAUDE.md

Guia de contexto para sessões de desenvolvimento neste repositório com o
Claude Code. Leia junto com `spec.md` (spec técnica completa) e `ROADMAP.md`
(ordem de execução).

## O que é este projeto

Fizzy Calendar Sync: extensão de Chrome (Manifest V3) que injeta um Side Panel
no Google Calendar mostrando cards abertos da Fizzy, permitindo agendá-los na
agenda por clique ou drag-and-drop. Distribuição pública na Chrome Web Store,
para qualquer usuário Google + conta Fizzy.

**Sempre consultar `spec.md` antes de implementar qualquer parte da integração
com API** — ele documenta correções já validadas contra a documentação oficial
da Fizzy e do Google que divergem do que seria intuitivo assumir.

## Decisões já tomadas (não reabrir sem motivo novo)

- Autenticação Fizzy é **Personal Access Token manual**, não OAuth. Não tentar
  implementar um fluxo OAuth para a Fizzy.
- Endpoint de cards exige `account_slug` no path:
  `GET /:account_slug/cards`. Nunca chamar `/cards` direto.
- Scope do Google é o mínimo necessário: `calendar.events` (não usar
  `calendar` completo nem `calendar.readonly` — o app só cria eventos, não lê
  a agenda).
- Permissão do manifest é `sidePanel` (camelCase), não `sidepanel`.
- Homepage oficial e política de privacidade do app vivem em `docs/` neste
  mesmo repo (GitHub Pages), **não** em ricsilva.com. Não sugerir mover isso
  para o site pessoal do autor.
- Agendamento duplicado do mesmo card é uma feature intencional, não um bug a
  "corrigir".
- **Sem suporte a Fizzy self-hosted.** Único host suportado: `app.fizzy.do`,
  hardcoded. Não implementar campo de URL configurável nem
  `optional_host_permissions` para isso.
- **Conflito de horário: sobreposição é permitida.** Não bloquear nem exigir
  confirmação extra ao agendar sobre um horário já ocupado.
- **`activeTab` não faz parte do manifest.** Não readicionar essa permissão.
- **`account_slug` é descoberto automaticamente**, chamando
  `GET https://app.fizzy.do/my/identity` logo após o usuário colar o Personal
  Access Token (a resposta traz a lista de contas com o campo `slug` de cada
  uma). Nunca pedir esse dado digitado manualmente.

## Convenções de código

- Preferir TypeScript se o setup do projeto permitir; se não, JS puro
  documentado.
- Toda chamada às APIs externas (Fizzy, Google Calendar) deve tratar:
  - erro de autenticação (401) → sinalizar estado "reconectar" na UI, nunca
    falhar silenciosamente.
  - rate limit (429, no caso da Fizzy) → retry com backoff exponencial.
  - paginação (header `Link: rel="next"` da Fizzy) → seguir até esgotar.
- `app.fizzy.do` é hardcoded como único host da Fizzy (self-hosted fora de
  escopo) — não criar campo de URL configurável para isso.
- Eventos criados no Google Calendar sempre devem incluir `timeZone` explícito
  em `start` e `end` (usar `Intl.DateTimeFormat().resolvedOptions().timeZone`
  do navegador do usuário).

## Segurança / permissões

- Antes de adicionar qualquer permissão nova ao `manifest.json`, justificar
  por que ela é necessária — permissões não usadas atrasam o review da Chrome
  Web Store. `activeTab` está sob suspeita de ser desnecessária (ver
  `spec.md` §7).
- Nunca commitar client secrets ou tokens de teste no repo.

## Onde as coisas vivem no repo (esperado)

```
/                  manifest.json, código-fonte da extensão
/docs              site estático (GitHub Pages): homepage + privacy policy do app
/spec.md           spec técnica (fonte da verdade de comportamento/API)
/ROADMAP.md         ordem de execução do projeto
/CLAUDE.md         este arquivo
```

## Etapas do projeto

Este projeto está dividido em duas etapas sequenciais — ver `ROADMAP.md` para
detalhes. Etapa 1 é o desenvolvimento da extensão em si. Etapa 2 é o que
depende da extensão já existir: as páginas em `docs/`, configuração do OAuth
consent screen do Google, e o listing da Chrome Web Store. **Não pular para a
Etapa 2 antes da Etapa 1 estar funcional**, já que o processo de verificação
OAuth do Google e o review da Chrome Web Store dependem de artefatos gerados
na Etapa 1 (ID fixo da extensão, permissões finais do manifest).
