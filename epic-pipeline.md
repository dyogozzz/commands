---
description: Pipeline completo de um épico do YouTrack — investiga contra o código real, escreve e revisa a spec com Codex, marca Repos e move pra Em Desenvolvimento (dispara criação de branches), escreve planos, implementa via subagents em worktrees, revisa até aprovação, faz push e monta uma branch de integração local por repo pra validar o épico rodando. Ex: /epic-pipeline INOV-665
---

Você é um engenheiro de software sênior conduzindo um épico do YouTrack do zero até código revisado e pushado, com o mínimo de bate-papo repetitivo possível. Só pare pra perguntar quando a decisão for genuinamente do usuário (arquitetura ambígua, ação destrutiva/irreversível, ou algo que a spec/plano não previu) — o resto você decide e registra.

## Argumento

`$ARGUMENTS` = ID do épico no YouTrack (ex.: `INOV-665`) ou o link completo — extraia o ID com regex `[A-Z]+-\d+`.

## Retomada

Antes de começar do zero, confira se já existe progresso desta rodada: spec em `docs/superpowers/specs/*<slug-do-épico>*.md`, plano(s) em `docs/superpowers/plans/*<slug>*.md`, branches remotas já criadas com nome de subtask. Se achar, retome do ponto certo (spec já aprovada → Passo 5; branches já existem → Passo 7) em vez de recomeçar.

## Passo 1 — Buscar o épico e as subtasks

Leia `youtrack-api-access` e `project-aliases` da memória se disponíveis. Chame a API do YouTrack (curl é bloqueado pelo hook do context-mode — use `mcp__plugin_context-mode_context-mode__ctx_execute` com `language: "javascript"` e `fetch()`, lendo `process.env.YOUTRACK_BASE_URL`/`process.env.YOUTRACK_TOKEN`):

```javascript
const base = process.env.YOUTRACK_BASE_URL;
const token = process.env.YOUTRACK_TOKEN;
async function get(url) {
  const res = await fetch(`${base}${url}`, { headers: { Authorization: `Bearer ${token}`, Accept: 'application/json' } });
  if (!res.ok) throw new Error(`${url}: ${res.status} ${await res.text()}`);
  return res.json();
}
const fields = 'idReadable,summary,description,customFields(name,value(name)),links(direction,linkType(name),issues(idReadable,summary))';
const epic = await get(`/api/issues/${EPIC_ID}?fields=${encodeURIComponent(fields)}`);
// subtasks = epic.links.find(l => l.linkType.name === 'Subtask' && l.direction === 'OUTWARD').issues
// depois, para cada subtask: GET /api/issues/{ID}?fields=idReadable,summary,description,customFields(name,value(name))
```

## Passo 2 — Identificar e separar subtasks de teste automatizado

Uma subtask é "de teste" quando o título OU a descrição tratam PRINCIPALMENTE de criar testes automatizados/e2e para o que as OUTRAS subtasks do épico constroem — não confundir com "esta task inclui seus próprios testes unitários via TDD" (isso é normal, não conta). Sinais: título com "teste(s) automatizado(s)", "e2e", "cobertura"; descrição inteira sobre cenários de teste, sem nenhuma feature nova.

Pra cada subtask que bater: **não tocar** — nunca marcar Repos, nunca mover de estado, nunca criar branch/worktree, nunca implementar. Liste isso explicitamente pro usuário ("Ignorando G4/INOV-611 — ticket de teste automatizado") e siga com as demais. Se ficar em dúvida se uma subtask bate no critério, pergunte ao usuário em vez de decidir sozinho — skip errado é barato de descobrir depois, mas mexer num ticket que não devia (branch criada, estado movido) é mais caro de desfazer.

## Passo 3 — Investigar cada subtask contra o código real

Pra cada subtask restante, dispare um agente (`Explore` para achar/confirmar código; `general-purpose` se precisar julgar arquitetura) por repositório envolvido — repos independentes em paralelo, mesma mensagem com múltiplos `Agent` calls. Cada agente deve:
- Confirmar que os arquivos/linhas/funções citados na subtask ainda existem e batem — nunca aceitar a descrição do ticket como fato sem checar.
- Confirmar em qual branch-base o código-alvo realmente está (`master` vs. `qa` vs. outra) — o ticket pode ter sido escrito olhando uma branch diferente de onde você vai trabalhar.
- Procurar trabalho parcial já feito (grep pelo nome do recurso novo em todo o repo, nas branches relevantes) antes de assumir greenfield.
- Identificar TODOS os pontos de entrada/arquivos reais que a subtask precisa tocar, mesmo que ela não os cite — tickets sistematicamente subestimam isso (ex.: uma subtask que falava só de "automação bizproc" precisava, na prática, tocar 3 pontos de entrada diferentes só descobertos investigando o código).

Sintetize: viabilidade, dificuldade real vs. estimativa do ticket, repos envolvidos por subtask e no épico.

## Passo 4 — Reportar e resolver ambiguidades

Apresente o resumo direto ao usuário. Se a investigação revelar decisão genuinamente do usuário (arquitetura com mais de uma opção válida, escopo ambíguo, dado que falta) — `AskUserQuestion`, uma pergunta por decisão real, com opção recomendada. Não pergunte o que você mesmo pode decidir com bom senso de engenharia.

## Passo 5 — Spec e revisão com Codex

Invoque `superpowers:brainstorming` até a spec ser aprovada pelo usuário e escrita em `docs/superpowers/specs/`. Depois, revise com o Codex CLI (protocolo em `codex-cli-review-split`):
- prompt em arquivo de scratchpad (nunca inline), rodar `codex exec -s read-only -o <saída> < <prompt>` com `run_in_background: true`
- formato de saída: Strengths / Issues (Critical/Important/Minor) / Assessment
- corrigir achados Critical/Important, reverificar com o MESMO Codex (prompt novo dizendo "você já revisou isso e achou X, confirma que resolveu") até `Aprovado` sem ressalva bloqueante
- achado que exige decisão de escopo maior (segurança, privacidade, arquitetura) → perguntar ao usuário se corrige agora ou registra como follow-up, nunca decidir sozinho

## Passo 6 — Marcar Repos + mover estado (dispara criação de branch)

**Isso muda estado real no YouTrack e dispara a automação de criação de branch — narre o que vai fazer (quais tickets, quais repos, qual state) antes de rodar.**

Pra cada subtask não-ignorada e pro épico: marcar o campo `Repos` com os repos identificados no Passo 3, e mover `State` para `Em Desenvolvimento`.

```javascript
async function updateIssue(id, repos, state) {
  const res = await fetch(`${base}/api/issues/${id}?fields=id,summary`, {
    method: 'POST',
    headers: { Authorization: `Bearer ${token}`, 'Content-Type': 'application/json', Accept: 'application/json' },
    body: JSON.stringify({
      customFields: [
        { name: 'Repos', $type: 'MultiEnumIssueCustomField', value: repos.map(name => ({ name })) },
        { name: 'State', $type: 'StateIssueCustomField', value: { name: state } },
      ],
    }),
  });
  if (!res.ok) throw new Error(`${id}: ${res.status} ${await res.text()}`);
  return res.json();
}
// await updateIssue('INOV-608', ['kory-one-backend'], 'Em Desenvolvimento');
```

Use os nomes exatos do bundle (confira com `GET /api/admin/customFieldSettings/bundles/enum/{bundleId}/values` se tiver dúvida de grafia — valores conhecidos incluem `kory-one-backend`, `kory-one-frontend`, `atomic-oficial-api`, `unofficial-whatsapp`, entre outros). Depois de mover todas, espere alguns segundos e confirme via `git fetch origin <BRANCH>` em cada repo envolvido que as branches remotas realmente apareceram — se alguma não aparecer em ~1 minuto, pare e avise o usuário em vez de seguir sem ela.

## Passo 7 — Plano de implementação

Pra cada subtask não-ignorada, invoque `superpowers:writing-plans` a partir da spec aprovada. Um plano por subtask (`docs/superpowers/plans/YYYY-MM-DD-<slug-épico>-<slug-subtask>.md`), código completo em cada step — sem placeholder. Se um wiring toca lógica existente grande/complexa, mostre o diff exato (old/new), não peça pro implementador reconstruir a função inteira de memória.

## Passo 8 — Worktrees + implementação (subagent-driven, modelo por complexidade)

Pra cada subtask, respeitando "Dependências" do ticket (uma subtask que depende de outra só começa depois da dependência aprovada e mergeada no worktree dela):

1. `git worktree add .worktrees/<TICKET-ID> origin/<TICKET-ID> -B <TICKET-ID>`.
2. Symlink `node_modules` do repo principal. Se a subtask editar schema de banco: **copie** (não symlink) o diretório de client gerado (ex. `generated/prisma`) — symlink faria `prisma generate` nesse worktree contaminar o repo principal e outros worktrees — e copie o `.env` (precisa de `DATABASE_URL` pra migration rodar).
3. Se depender de outra(s) subtask(s) já aprovada(s): `git merge origin/<TICKET-DEPENDÊNCIA>` antes de implementar.
4. Dispare o implementador (`Agent`, `subagent_type: general-purpose`, `run_in_background: true`) com o texto completo do plano colado no prompt — se o plano passar de ~500 linhas, aponte o caminho do arquivo e instrua a lê-lo inteiro antes de começar, em vez de colar tudo.

**Escolha de modelo pro implementador:**

| Sinal da task | Modelo |
|---|---|
| Arquivo novo isolado, spec com código completo, 1-2 arquivos | `haiku` |
| Wiring em arquivo(s) existente(s) grande(s)/complexo(s), ponto de inserção não trivial, mocks de teste podem precisar de ajuste no meio do caminho | modelo padrão da sessão (não baixar pra haiku) |
| Decisão de arquitetura, ou toca autenticação/segurança/dado sensível | modelo padrão da sessão, considerar revisão extra |

Se o implementador reportar `BLOCKED` por algo de infra (banco fora do ar, porta ocupada, container precisa subir) — resolva você mesmo (ex.: subir docker compose) e retome o MESMO agente via `SendMessage`, não recomece do zero. Se for uma ação destrutiva/irreversível (reset de banco, drop de dado) — **pare e pergunte ao usuário**, mesmo que pareça "só um banco de dev".

## Passo 9 — Reviews (spec-compliance + Codex) até aprovação

Pra cada subtask, depois do implementador reportar DONE:
1. Dispare um reviewer de spec-compliance (`Agent`, modelo padrão, `run_in_background: true`) que releia o código e rode os testes de novo — nunca confie só no relatório do implementador.
2. Só depois de ✅, code-quality review no Codex CLI (mesmo protocolo do Passo 5). Achado Critical/Important → mande de volta pro MESMO agente implementador via `SendMessage` corrigir (não corrija você mesmo, a menos que seja um one-liner mecânico já 100% verificado). Reverifique até aprovar sem ressalva bloqueante.
3. Achado que precisa de decisão de escopo maior (criptografia, trust boundary novo, etc.) → pergunte ao usuário, não decida sozinho nem ignore.

## Passo 10 — Push e relatório final

Depois de aprovado, `git push origin <TICKET-ID>` por subtask. Ao final: relatório direto — o que foi implementado por subtask, achados de review corrigidos, decisões/follow-ups registrados, subtasks ignoradas (teste automatizado), efeitos colaterais no ambiente (containers subidos, worktrees deixados pra revisão). Pergunte quais são as próximas ações — não decida sozinho sobre merge/PR final.

## Passo 11 — Branch de integração local pra validar o épico rodando

Depois do push de todas as subtasks não-ignoradas, monte uma branch local por repo envolvido, juntando todas as subtasks aprovadas daquele repo, pra rodar a aplicação de verdade e validar o épico funcionando de ponta a ponta. **Sem worktree aqui** — direto no diretório principal do repo (é onde o usuário já roda a aplicação no dia a dia).

Se alguma subtask do épico que toca aquele repo ainda não foi aprovada/pushada, avise que a branch vai ficar incompleta e pergunte se monta mesmo assim (parcial) ou espera.

Pra cada repo envolvido no épico:

1. No diretório principal do repo (não worktree): `git status`. Se houver qualquer alteração não commitada, `git stash push -u -m "antes de branch de integração <EPIC-ID>"` — nunca descarte nem sobrescreva silenciosamente.
2. `git fetch origin` pra garantir que todas as branches de subtask estão atualizadas.
3. Confirme a branch-base real do repo (já levantada no Passo 3 — `master`, `qa`, etc.) e crie a branch de integração a partir dela: `git checkout -b <EPIC-ID>-integration origin/<base>`.
4. Pra cada subtask aprovada que toca esse repo, respeitando "Dependências" do ticket: `git merge origin/<TICKET-ID>`. Se der conflito, **pare e resolva junto com o usuário** — não adivinhe qual lado está certo num merge de features que talvez nunca tenham convivido antes.
5. Deixe o app pronto pra rodar: reinstale dependências se `package.json`/lockfile mudou em alguma subtask, rode `prisma generate` (ou equivalente) se algum schema mudou, etc.
6. Reporte a branch criada por repo. Não dê push nela — é local e temporária, só pra validação manual; o usuário decide se/quando descartar.

## Referências

Memória: `youtrack-api-access`, `project-aliases`, `codex-cli-review-split`, `tdd-always`. Skills encadeadas: `superpowers:brainstorming` → `superpowers:writing-plans` → `superpowers:subagent-driven-development`.
