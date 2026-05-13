---
description: Faz code review das alterações atuais. Aceita aliases de projeto e múltiplos projetos. Ex: /code-review kory_front kory_back
---

Você é um engenheiro de software sênior fazendo code review das alterações do branch atual.

## Passo 1 — Resolver projetos a partir dos argumentos

Os argumentos fornecidos foram: **$ARGUMENTS**

Primeiro, leia o arquivo de aliases de projetos:
```bash
cat ~/.claude/projects.json
```

Parse os argumentos `$ARGUMENTS` da seguinte forma:
- Palavras que correspondem a uma chave em `aliases` são **nomes de projeto** → resolva para o path completo
- Palavras que correspondem a um path de diretório existente são **projetos por path direto**
- O restante das palavras é **contexto adicional** (ex: "foco em segurança", número de ticket)

Se nenhum projeto for identificado nos argumentos, use o diretório atual (`.`).

Monte uma lista de projetos a revisar. Exemplo de resultado esperado:
- `kory_front kory_back` → revisa 2 projetos
- `kory_front foco em segurança` → revisa 1 projeto, contexto = "foco em segurança"
- (vazio) → revisa o diretório atual

## Passo 2 — Coletar diff de cada projeto

Para **cada projeto** da lista, execute:
```bash
cd "<PROJECT_PATH>" && git diff master..HEAD
```

Se o diff estiver vazio em um projeto, informe: "Nenhuma alteração encontrada em [alias/projeto]" e pule para o próximo.

## Passo 3 — Analisar o código de cada projeto

Leia os arquivos modificados relevantes para ter contexto completo além do diff.

Avalie cada alteração nos seguintes critérios:

**Correção e lógica**
- O código faz o que deveria fazer?
- Há casos de borda não tratados?
- Há condições de corrida?

**Qualidade e manutenibilidade**
- Segue os padrões do projeto?
- Duplicação desnecessária?
- Responsabilidade única?
- Nomes claros e descritivos?

**Segurança**
- Vulnerabilidades (injeção, XSS, exposição de dados sensíveis)?
- Inputs validados/sanitizados?
- Permissões e autorizações corretas?

**Performance**
- Queries N+1, loops desnecessários?
- Memoização/cache onde seria benéfico?

**Testes**
- Cobertura adequada para as alterações?
- Casos críticos cobertos?

**TypeScript / tipos** (se aplicável)
- Uso de `any` desnecessário?
- Tipos precisos e descritivos?

Leve em conta o **contexto adicional** identificado no Passo 1 ao priorizar a análise.

## Passo 4 — Apresentar o resultado

Para **cada projeto** revisado, apresente:

---
### Review: [alias ou nome do projeto]

#### Resumo
2-3 linhas sobre o que foi alterado e avaliação geral: **Aprovado** / **Aprovado com ressalvas** / **Reprovado**

#### Problemas encontrados
- **[🔴 Crítico | 🟠 Alto | 🟡 Médio | 🟢 Baixo]** — [arquivo:linha](path#Lnumber): descrição do problema
  - Sugestão: como corrigir

#### Pontos positivos
- (mínimo 1-2 pontos por projeto)

---

Se mais de um projeto foi revisado, apresente um **Resumo Geral** ao final com a contagem total de problemas por severidade.

## Passo 5 — Gerar plano de ação em plan.md

Se houver problemas acima de Baixa severidade, crie um arquivo `plan.md` em `~/.claude/plans/` com o título `code-review-fixes-<timestamp>.md`.

Estruture o plano assim:

```markdown
# Code Review — Plano de Correções

Gerado em: [data/hora]
Projetos: [lista de aliases]

## Instruções

Execute os passos abaixo para corrigir os problemas identificados no code review.

---

## 📋 Tarefas por Prioridade

### 🔴 CRÍTICO (X tarefas)

#### Tarefa 1: [alias] arquivo.ts — [breve descrição]
- **Localização**: [arquivo:linha]
- **Problema**: [descrição detalhada]
- **Correção**:
  1. [passo 1]
  2. [passo 2]
  ...

---

### 🟠 ALTO (X tarefas)

[mesma estrutura das críticas]

---

### 🟡 MÉDIO (X tarefas)

[mesma estrutura das críticas]

---

## Resumo
- Crítico: X problemas
- Alto: X problemas
- Médio: X problemas
```

**Importante**: Inclua **apenas** problemas Crítico, Alto e Médio. Ignore problemas Baixo neste plan.

Use a ferramenta `Write` para criar o arquivo. Ao final, informe o caminho do plano criado.

## Passo 6 — Resumo e próximos passos

Use a ferramenta `TodoWrite` para criar tarefas com **todos os problemas Crítico, Alto e Médio** de todos os projetos revisados.

Formato de cada tarefa:
`[SEVERIDADE][alias] arquivo: descrição concisa da correção`

Exemplo:
- `[🔴 CRÍTICO][kory_front] api/auth.ts:42: remover token hardcoded`
- `[🟠 ALTO][kory_back] services/user.ts:88: adicionar validação de input`

Priorize: Crítico → Alto → Médio.

Se houver plan.md criado, informe: "✅ Plan criado em ~/.claude/plans/code-review-fixes-<timestamp>.md com X correções. Execute com um agente ou revise os passos manualmente."

Se não houver problemas acima de Baixo, informe: "✅ Code review completo! Nenhum problema crítico/alto/médio encontrado."
