---
description: Documenta alterações, cria commits e comenta no YouTrack com resumo end-to-end. Ex: /commit-doc kory_front kory_back
---

Você é um engenheiro de software sênior documentando e commitando alterações do time.

## Passo 1 — Resolver projetos

Os argumentos fornecidos foram: **$ARGUMENTS**

Leia o arquivo de aliases:
```bash
cat ~/.claude/projects.json
```

Parse `$ARGUMENTS`:
- Palavras que batem com uma chave em `aliases` → resolva para o path completo
- Palavras que correspondem a um path de diretório existente → usar como está
- Se nenhum projeto identificado → usar diretório atual (`.`)

## Passo 2 — Coletar alterações de cada projeto

Para cada projeto, execute:
```bash
cd "<PROJECT_PATH>" && git branch --show-current && git diff master..HEAD
```

- Se não houver alterações em um projeto, informe "Nenhuma alteração em [alias]" e pule.
- Extraia o **ticket ID do YouTrack** do nome da branch com o padrão `[A-Z]+-\d+`
  (ex: `KOR-123` de `KOR-123-fix-avatar` ou `KOR-123` direto)
- Se nenhum ticket for encontrado em nenhuma branch, avise o usuário e pergunte se deseja continuar apenas com os commits.

## Passo 3 — Ler arquivos modificados

Para cada projeto com alterações, leia os arquivos modificados para entender o contexto completo das mudanças além do diff.

## Passo 4 — Commitar cada projeto

Para cada projeto com alterações:

1. Stage todos os arquivos modificados:
```bash
cd "<PROJECT_PATH>" && git add -A
```

2. Gere uma commit message concisa e objetiva no formato: `<tipo>: <descrição curta>`
   - Tipos: `fix`, `feat`, `refactor`, `chore`, `style`, `docs`
   - Máximo de 72 caracteres, em português ou inglês conforme padrão do projeto

3. Crie o commit:
```bash
cd "<PROJECT_PATH>" && git commit -m "<mensagem>"
```

Informe o resultado de cada commit (hash + mensagem).

## Passo 5 — Montar resumo end-to-end para o YouTrack

Monte o comentário em duas seções obrigatórias:

**Seção 1 — O que foi alterado**

Descreva de forma objetiva o que mudou no comportamento do sistema — não o que foi feito no código, mas o que o usuário/sistema passa a ver diferente. Cubra todos os projetos alterados de forma integrada (visão end-to-end).

Inclua ao final uma listagem por projeto:
- **[alias]** · `[hash curto]`: [arquivos modificados] — [1 linha do que mudou]

**Seção 2 — Como testar**

Liste os passos práticos para validar que as alterações funcionam corretamente. Seja específico: quais telas, ações, endpoints, casos de borda ou fluxos devem ser verificados. Inclua o resultado esperado para cada passo.

Formato final do comentário:

```
## O que foi alterado

[Descrição objetiva do comportamento novo/corrigido, em 3-5 linhas]

[alias] · hash: arquivos — resumo
[alias] · hash: arquivos — resumo

---

## Como testar

1. [Passo concreto] → [resultado esperado]
2. [Passo concreto] → [resultado esperado]
...
```

Se não houver passos de teste óbvios (ex: refactor interno), escreva "Sem impacto funcional visível — validar via testes automatizados."

## Passo 6 — Atualizar FAQ (apenas INOV- em projetos kory)

Para **cada ticket INOV-** encontrado nos projetos **kory_front ou kory_back**:

1. Verifique se o ticket começa com `INOV-` (não é SUST-)
2. Se sim, e se está em um projeto kory, execute:
```bash
# Ler as mudanças da feature para contextualizar o FAQ
git log --grep="$TICKET_ID" -1 --pretty=format:"%B"
```

3. Chame a skill de FAQ:
```
/faq-update $TICKET_ID
```

**Exemplos:**
- `INOV-123` em `kory_front` → **atualiza FAQ** ✅
- `INOV-456` em `atomic` → **ignora** (não é kory)
- `SUST-789` em `kory_back` → **ignora** (é sustentação, não feature)

Se houver erro ao atualizar FAQ, informe o usuário mas **não interrompa o fluxo de commits**.

## Passo 7 — Comentar no YouTrack

Agrupe os projetos por ticket ID. Para cada ticket identificado, poste o resumo via API.

Salve o body JSON em um arquivo temporário para evitar problemas de escaping:

```bash
cat > /tmp/yt_comment.json << 'ENDJSON'
{"text": "<RESUMO_AQUI>"}
ENDJSON

curl -s -w "\n%{http_code}" -X POST \
  "$YOUTRACK_BASE_URL/api/issues/<TICKET_ID>/comments" \
  -H "Authorization: Bearer $YOUTRACK_TOKEN" \
  -H "Content-Type: application/json" \
  -H "Accept: application/json" \
  -d @/tmp/yt_comment.json
```

- Se o HTTP status for `200` ou `201`: informe "✅ Comentário postado no ticket \<ID\>"
- Se retornar erro: exiba o status e a mensagem completa de erro para o usuário
- Ao final, remova o arquivo temporário: `rm -f /tmp/yt_comment.json`

**Importante:** As variáveis `$YOUTRACK_BASE_URL` e `$YOUTRACK_TOKEN` vêm do ambiente — não as solicite ao usuário nem as exiba no output.
