---
description: Cria/atualiza FAQ no Notion sobre features. Ex: /faq-update INOV-123 "Descrição da feature"
---

Você é um engenheiro de software atualizando a documentação de features no Notion.

## Passo 1 — Validar tipo de ticket

O ticket fornecido foi: **$ARGUMENTS**

Parse o ticket ID:
- Se começa com `INOV-` → Feature de inovação, **continua com atualização do FAQ**
- Se começa com `SUST-` → Bug fix/sustentação, **avisa que não precisa FAQ e termina**
- Se outro padrão → avisa para confirmar o tipo (INOV ou SUST)

## Passo 2 — Buscar descrição da task no YouTrack (se INOV)

Se for inovação, busque o contexto da task:

```bash
curl -s -X GET \
  "$YOUTRACK_BASE_URL/api/issues/$TICKET_ID?fields=idReadable,summary,description,customFields" \
  -H "Authorization: Bearer $YOUTRACK_TOKEN" \
  -H "Content-Type: application/json" | jq '.'
```

Extraia:
- **Título** — campo `summary`
- **Descrição da task** — campo `description` (contexto da feature)
- **Custom fields** — qualquer informação adicional relevante

## Passo 3 — Coletar informações de código e commits (se INOV)

Combine a descrição da task com análise do código:

```bash
cd . && git log --grep="$TICKET_ID" -1 --pretty=format:"%B"
```

Extraia:
- **O que foi alterado** — mudanças técnicas (do commit)
- **Impacto** — como isso afeta o usuário/sistema? (combine descrição task + código)
- **Como usar** — Como usar a feature? (descreva baseado em código + documentação da task)
- **Exemplos técnicos** — se aplicável, inclua exemplos de uso baseado no código

**Nota:** Se você já fez `/code-review` ou `/commit-doc` antes, reutilize a análise de código do contexto anterior para economizar tokens.

## Passo 4 — Verificar se página existe no Notion

Via API REST do Notion:
```bash
curl -s -X POST https://api.notion.com/v1/databases/$NOTION_DATABASE_ID/query \
  -H "Authorization: Bearer $NOTION_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"filter": {"property": "Ticket", "title": {"equals": "$TICKET_ID"}}}'
```

- Se existir → vá para Passo 5 (atualizar)
- Se não existir → vá para Passo 6 (criar nova)

## Passo 5 — Atualizar página existente (se já existe)

Obtenha o `page_id` da resposta anterior e atualize com novos campos.

Campos a atualizar:
- **Descrição** — (append ou replace)
- **Data de atualização** — data atual
- **Versão** — incrementar minor (ex: v1.0 → v1.1)

Estrutura JSON mínima:
```json
{
  "properties": {
    "Descrição": {
      "rich_text": [{"type": "text", "text": {"content": "<nova descrição>"}}]
    },
    "Atualizado em": {
      "date": {"start": "YYYY-MM-DD"}
    }
  }
}
```

## Passo 6 — Criar nova página (se não existe)

Via API:
```bash
curl -s -X POST https://api.notion.com/v1/pages \
  -H "Authorization: Bearer $NOTION_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "parent": {"database_id": "$NOTION_DATABASE_ID"},
    "properties": {
      "Ticket": {"title": [{"text": {"content": "$TICKET_ID"}}]},
      "Feature": {"rich_text": [{"text": {"content": "$FEATURE_TITLE"}}]},
      "Descrição": {"rich_text": [{"text": {"content": "$FEATURE_DESCRIPTION"}}]},
      "Impacto": {"rich_text": [{"text": {"content": "$IMPACT"}}]},
      "Como usar": {"rich_text": [{"text": {"content": "$HOW_TO_USE"}}]},
      "Status": {"select": {"name": "Ativo"}},
      "Criado em": {"date": {"start": "YYYY-MM-DD"}},
      "Atualizado em": {"date": {"start": "YYYY-MM-DD"}}
    }
  }'
```

## Passo 7 — Confirmar resultado

Se INOV:
- ✅ "FAQ atualizado no Notion: https://notion.so/<page-id>"

Se SUST:
- ℹ️ "Ticket SUST- identificado. FAQs não são atualizados para bug fixes/sustentação."

Se erro:
- ❌ Mostre o erro da API com contexto
