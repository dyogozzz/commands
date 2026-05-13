---
description: Documenta alterações já commitadas (FAQ + YouTrack). Ex: /doc-only INOV-123 kory_front
---

Você é um engenheiro de software documentando alterações já commitadas.

## Passo 1 — Resolver ticket e projeto

Os argumentos fornecidos foram: **$ARGUMENTS**

Parse:
- Primeiro argumento = **Ticket ID** (ex: INOV-123, SUST-456)
- Argumentos restantes = **projetos** (aliases ou paths)
  - Se vazio, usa o diretório atual

Exemplo:
- `INOV-123 kory_front` → Ticket INOV-123, projeto kory_front
- `SUST-456 kory_back` → Ticket SUST-456, projeto kory_back
- `INOV-789` → Ticket INOV-789, usa diretório atual

## Passo 2 — Coletar alterações do ticket

Para cada projeto, execute:
```bash
cd "<PROJECT_PATH>" && git log --grep="$TICKET_ID" --oneline && git diff master..HEAD
```

- Se não encontrar commits com o ticket ID no nome, avise o usuário
- Se não houver diff em relação a master, avise que não há mudanças não mergeadas ainda

## Passo 3 — Ler arquivos modificados

Leia os arquivos modificados para ter contexto completo além do diff.

## Passo 4 — Montar resumo end-to-end para o YouTrack

Monte o comentário em duas seções obrigatórias:

**Seção 1 — O que foi alterado**

Descreva de forma objetiva o que mudou no comportamento do sistema.

Inclua ao final uma listagem por projeto:
- **[alias]** · `[hash curto]`: [arquivos modificados] — [1 linha do que mudou]

**Seção 2 — Como testar**

Liste os passos práticos para validar que as alterações funcionam corretamente.

Formato final do comentário:

```
## O que foi alterado

[Descrição objetiva do comportamento novo/corrigido, em 3-5 linhas]

[alias] · hash: arquivos — resumo

---

## Como testar

1. [Passo concreto] → [resultado esperado]
2. [Passo concreto] → [resultado esperado]
...
```

Se não houver passos de teste óbvios (ex: refactor interno), escreva "Sem impacto funcional visível — validar via testes automatizados."

## Passo 5 — Atualizar FAQ (apenas INOV- em projetos kory)

Para **INOV-** em **kory_front ou kory_back**:

1. Verifique se o ticket começa com `INOV-`
2. Se sim, execute:
```bash
git log --grep="$TICKET_ID" -1 --pretty=format:"%B"
```

3. Chame a skill de FAQ:
```
/faq-update $TICKET_ID
```

**Exemplos:**
- `INOV-123` em `kory_front` → **atualiza FAQ** ✅
- `INOV-456` em `atomic` → **ignora** (não é kory)
- `SUST-789` em `kory_back` → **ignora** (é sustentação)

Se houver erro ao atualizar FAQ, informe o usuário mas **não interrompa o fluxo**.

## Passo 6 — Comentar no YouTrack

Se encontrou o ticket ID, poste o resumo via API:

```bash
cat > /tmp/yt_comment.json << 'ENDJSON'
{"text": "<RESUMO_AQUI>"}
ENDJSON

curl -s -w "\n%{http_code}" -X POST \
  "$YOUTRACK_BASE_URL/api/issues/$TICKET_ID/comments" \
  -H "Authorization: Bearer $YOUTRACK_TOKEN" \
  -H "Content-Type: application/json" \
  -H "Accept: application/json" \
  -d @/tmp/yt_comment.json
```

- Se HTTP status `200` ou `201`: informe "✅ Comentário postado no ticket <ID>"
- Se erro: exiba status e mensagem de erro
- Remova o arquivo temporário: `rm -f /tmp/yt_comment.json`

**Importante:** As variáveis `$YOUTRACK_BASE_URL` e `$YOUTRACK_TOKEN` vêm do ambiente.

## Passo 7 — Resumo final

Informe:
- ✅ FAQ atualizado (se INOV-)
- ✅ Comentário postado no YouTrack
- 📝 Resumo do que foi documentado
