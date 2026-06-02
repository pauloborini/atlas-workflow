# PLAN ATLAS-V030 — Refatoração v0.2.0 → v0.3.0 (família única, validator subagent, paths canônicos)

| Campo | Valor |
|-------|-------|
| **PRD** | Decisões D1, D2, ConfigKill, S1, S2, S3, S4, S5, S6', S7, S8 travadas em chat (não há `PRD_*.md` em disco — referenciar diretamente este §2 como invariantes) |
| **Package / app** | `packages/mcp-server/`, `packages/skills/`, `packages/orchestrator/`, `packages/templates/`, `hooks/claude/`, `plugin-manifests/`, raiz do plugin |
| **Tipo** | `refactoring` (breaking — colapso de famílias, remoção de tool MCP, troca de paths canônicos) |
| **execution_mode** | `orchestrated-per-slice` |
| **Data** | 2026-06-01 |

**Escopo técnico:** colapsar 3 famílias (claude/cursor/codex) em família única `atlas-*`; remover orquestrador de família (`atlas_lock_family`, param `family`); registrar `atlas-task-validator` como subagent determinístico; padronizar paths `.atlas/plans/` + `.atlas/state/`; injetar boundary automático executor→validator; promover `atlas_run_state` MCP a SSoT de estado; espelhamento declarativo plano→todo nativo; gate `--review` explícito; veredito JSON estruturado; limpeza legado `§14`; README "Como funciona"; bump versão `v0.3.0`.

**Fora:** novas skills além das 7 finais; criação de PRD formal em disco; alteração de templates de PRD/PLAN além do que `§14→§8` exige; commit/push/tag/deploy (somente edição local); arquivos `raycast/`, `archive/v0.1.10/`.

Política: [BOUNDARY_PRD_PLAN.md](../../packages/templates/BOUNDARY_PRD_PLAN.md).

---

## Metadados de execução
- Plan prefix: `codex`
- Execution mode: `orchestrated-per-slice`
- Executor skill: `codex-plan-execute` (single executor pós-refactor: `atlas-plan-execute`)
- Internal validator: `codex-task-validator` (single validator pós-refactor: `atlas-task-validator`)
- External review: `codex-slice-review` (optional, via flag `--review`)

**Justificativa do modo orchestrated-per-slice:** refactor multi-camada (MCP server JS, ~7 SKILL.md, orchestrator config, hooks JS, plugin manifest, README). Slices independentes reduzem blast radius — falha em uma slice (ex.: MCP server) não derruba refactor textual de skills. Checks por task: build node + smoke `atlas_ping`. Fechamento de slice: validator subagent cold review do diff. Parar em `blocked` se quebrar smoke `atlas_ping` ou validar artefato canônico falhar.

---

## 1. Tradução executiva

Refatoração estrutural do plugin Atlas Workflow Orchestrator. Hoje plugin empacota 21 skills fan-out em 3 famílias (`claude-*`, `cursor-*`, `codex-*`) com orquestrador MCP roteando família via `atlas_lock_family` + param `family` em `atlas_preflight`/`atlas_lock_dispatch`. Validator é skill in-band (sem isolamento de contexto). Paths variam entre `.cursor/plans/` e `.codex/plans/`. Veredito do validator é prosa livre. Estado persistido por file IO direto. Resultado pós-refactor: 7 skills `atlas-*` invocáveis direto sem roteamento; validator como `subagent_type` real; paths canônicos `.atlas/plans/` + `.atlas/state/<run_id>/`; veredito JSON; estado via MCP `atlas_run_state`; plano espelhado automaticamente no todo nativo do cliente; review opcional só por flag.

**Padrão de referência no monorepo:** estrutura atual de `packages/skills/atlas-*/SKILL.md` já usa nomes finais — refatoração é principalmente conteúdo (substituir `codex-*` por `atlas-*` no corpo dos SKILL.md) + MCP server (remover dimensão família) + manifest (descrição + versão).

**Diferenças obrigatórias vs estado atual**

| Tema | Estado atual (v0.2.0) | Esta entrega (v0.3.0) |
|------|----------------------|------------------------|
| Famílias | 3 (`claude-*`, `cursor-*`, `codex-*`), fan-out no bundle | 1 (`atlas-*`), sem fan-out |
| Variante orchestrated | `*-plan-execute-orchestrated` separada | Comportamento interno default de `atlas-plan-execute` |
| Validator | Skill in-band | Subagent `subagent_type: atlas-task-validator` |
| Boundary executor→validator | Prompt manual | State file `.atlas/state/<run_id>/<slice>.json` |
| Paths plano | `.cursor/plans/` ou `.codex/plans/` | `.atlas/plans/` (migration order: `.atlas/` → `.cursor/` → `.codex/` por 1 release) |
| Config família | `orchestrator/atlas_workflows_config.md` | Deletado |
| MCP `atlas_lock_family` | Existe | Deletado |
| Param `family` em `atlas_preflight` / `atlas_lock_dispatch` | Obrigatório | Removido |
| Veredito validator | Texto livre | JSON `{verdict, findings, observations, boundary_violations}` |
| Estado | File IO direto em `.atlas-run/` | MCP `atlas_run_state` SSoT |
| Slice review trigger | Heurística + flag | Apenas flag `--review` |
| Todo runtime | Não declarado | Espelhamento declarativo plano→todo nativo (TodoWrite/todo_write/tasks) |
| `§14` legado | Espalhado em skills/templates | Substituído por `§8` |
| Versão | `0.2.0` | `0.3.0` (breaking) |

Capacidades já existentes (não reimplementar):
- `atlas_run_state` MCP tool já implementada em `packages/mcp-server/server.js` (linhas ~258, ~286, ~335, ~381 — só promover uso nas skills).
- Skills `atlas-*` já existem em `packages/skills/atlas-*/SKILL.md` — refactor é de conteúdo, não criação.
- State dir `.atlas-run/` já existe — migrar para `.atlas/state/` mantendo schema.

---

## 2. Invariantes de execução

- **I1 — Família única:** após esta entrega, nenhum identificador `claude-*`, `cursor-*` ou `codex-*` referente a skill da cadeia Atlas pode aparecer em código, manifest, hooks ou documentação. Exceções permitidas: comentário de migration histórica em README; `.github/workflows/` referenciando build steps independentes.
- **I2 — Sem variante orchestrated:** identificador `*-plan-execute-orchestrated` banido em qualquer arquivo distribuído.
- **I3 — Validator isolado:** `atlas-task-validator` é invocável como `subagent_type` real; chamada do executor passa apenas `state_path`, nunca colando contrato/diff no prompt.
- **I4 — State file canônico:** `.atlas/state/<run_id>/<slice>.json` é o boundary single-direction executor→validator. Schema mínimo: `{slice, tasks, files_changed, diff_stat, plan_path, boundary_refs}`.
- **I5 — Paths canônicos:** novos artefatos vão em `.atlas/plans/` e `.atlas/state/`. Skill `atlas-plan-handoff` aceita leitura em `.cursor/plans/` e `.codex/plans/` por 1 release com deprecation warning; escrita só em `.atlas/plans/`.
- **I6 — Veredito JSON:** validator retorna JSON estrito com chaves `verdict` ∈ `{pass, fail, pass_with_observations}`, `findings[]`, `observations[]`, `boundary_violations[]`. Executor decide repair vs done por parse de campo, nunca por substring de prosa.
- **I7 — `atlas_run_state` SSoT:** skills leem/escrevem estado via tool MCP, não file IO direto. File IO de fallback só se MCP indisponível, e nesse caso skill avisa e aborta gate.
- **I8 — Espelhamento todo:** ao entrar em `implementing`, executor espelha tasks do plano (IDs `T1.1`, `T1.2`...) no todo nativo do cliente. Plano = SSoT. Estados: `pending` (`ready`) → `in_progress` (`implementing`/`gating`) → `completed` (`task_done`). Divergência ⇒ sincroniza do plano pro todo.
- **I9 — Gate `--review` explícito:** `atlas-slice-review` só dispara se flag `--review` presente no comando do usuário ou nos argumentos passados ao executor. Sem auto-trigger heurístico.
- **I10 — Smoke `atlas_ping` verde:** após qualquer slice tocar `packages/mcp-server/server.js`, smoke `atlas_ping` precisa retornar `ok` com `version: 0.3.0` antes de fechar a slice.
- **I11 — Sem deploy/commit:** nenhuma task desta execução roda `git commit`, `git push`, `git tag`, scripts de deploy ou modifica `archive/v0.1.10/` e `raycast/`.

---

## 3. Pitfalls

- Substituição cega `codex-*` → `atlas-*` em diretórios `archive/` ou `raycast/` → **não tocar arquivos legados/archive**.
- Esquecer de atualizar `description` do `plugin-manifests/claude/plugin.json` que menciona "21 skills (claude/cursor/codex)" → **fica falso pós-refactor**.
- Remover dimensão `family` do MCP server quebra clientes v0.2.0 instalados → **assumir breaking; bump major-ish (`0.3.0`); CHANGELOG.md sinalizar**.
- Migrar paths `.cursor/plans/` direto sem leitura de fallback → **handoff perde planos legados em repos consumidores**.
- Validator subagent registrado mas executor ainda passa contrato inline no prompt → **perde determinismo prometido em I3**.
- State file gravado pelo executor sem `run_id` consistente entre slices → **MCP `atlas_run_state` perde rastro**.
- Espelhamento todo invertido (atualiza todo e nunca plano) → **plano deixa de ser SSoT; viola I8**.
- Veredito JSON parseado por regex em vez de `JSON.parse` → **frágil; usar parser estrito**.
- Manter `§14` em qualquer string visível (mesmo comentário) → **confunde leitor novo; deve ir tudo a `§8`**.

---

## 4. Estado na abertura da sprint (pré-implementação)

- `packages/skills/atlas-*/SKILL.md` existem (7 dirs), mas conteúdo interno ainda usa identificadores `codex-*` (frontmatter `name:`, descrição, prosa) — refactor é principalmente textual.
- `packages/mcp-server/server.js` tem ~1700 linhas, hardcoda 3 famílias (`claude|cursor|codex`) em regex (linha 140), expõe tool `atlas_lock_family` (linha 263, 1530), assertiva `assertDispatchFamily` (linha 1139). Tools `atlas_run_state`, `atlas_preflight`, `atlas_lock_dispatch` existem mas com dimensão família embutida.
- `packages/orchestrator/atlas_workflows_config.md` (17KB) mapeia famílias→skills e sequências full/direct/interview-only. Será deletado integralmente.
- `packages/templates/PLAN_TEMPLATE.md` + `BOUNDARY_PRD_PLAN.md` já estão neutros em termos de família. `PLAN_TEMPLATE.md` tem nota `(§14)` opcional residual.
- `plugin-manifests/claude/plugin.json` e `plugin-manifests/codex/plugin.json` declaram "21 skills (claude/cursor/codex)" na `description` — precisa atualizar.
- `hooks/claude/atlas-workflow-hook.js` existe mas precisa inspeção para verificar se referencia famílias.
- `VERSION` = `0.2.0`. Bump → `0.3.0`.
- `.atlas-run/` existe em repo; será migrado conceitualmente para `.atlas/state/` (mas dir físico em repos consumidores). No próprio repo do plugin não há `.atlas-run/<run_id>` ativo.

---

## 5. Tarefas de execução

### Slice A — Naming + Legado (D1 + S4)

#### T01. Auditoria de identificadores legado

- **Objetivo:** mapa completo de ocorrências `codex-*`, `cursor-*`, `claude-*`, `*-orchestrated`, `§14` em arquivos distribuíveis (excluindo `archive/`, `raycast/`, `.git/`).
- **Pré-condições:** nenhuma
- **Mudança esperada:** arquivo temporário `.atlas/state/<run_id>/A-audit.txt` com saída de `grep -rn` agrupada por arquivo. Não é artefato distribuído — vive em `.atlas/state/` por isso fica gitignored implícito.
- **Invariantes preservados:** I1, I2
- **Não mudar:** nenhum arquivo de produção nesta task
- **Não fazer:** editar antes de auditar — base para T02..T05
- **Dependências:** nenhuma
- **Critério de done:** arquivo de auditoria gravado; contagem de ocorrências por categoria reportada no log
- **Validação local:**
  ```bash
  cd /Volumes/Dados/projetos/atlas-workflow && \
    grep -rn --include='*.md' --include='*.js' --include='*.json' \
    -E '(codex-|cursor-|claude-)(plan-|sprint-|task-|slice-|prd-|direct-)|§14|plan-execute-orchestrated' \
    packages/ hooks/ plugin-manifests/ README.md VERSION 2>/dev/null | tee .atlas/state/<run_id>/A-audit.txt | wc -l
  ```

#### T02. Renomear identificadores nas 7 skills `atlas-*`

- **Objetivo:** SKILL.md de `atlas-direct-execute`, `atlas-plan-execute`, `atlas-plan-handoff`, `atlas-prd-interview`, `atlas-slice-review`, `atlas-sprint-prd-generator`, `atlas-task-validator` passam a referir-se apenas a `atlas-*` no frontmatter `name:`, `description:`, prosa, exemplos.
- **Referência:** `packages/skills/atlas-plan-execute/SKILL.md` (linhas 3, 60 já mapeadas — extrapolar)
- **Pré-condições:** T01 concluída
- **Mudança esperada:** 7 arquivos `SKILL.md` sem ocorrência de `codex-`, `cursor-`, `claude-` em contexto de skill (manter eventual menção a "Claude Code"/"Cursor"/"Codex" como cliente em prosa explicativa apenas se necessário e claramente rotulado como cliente, não skill).
- **Invariantes preservados:** I1
- **Não mudar:** arquivos em `archive/`, `raycast/`
- **Não fazer:** rename de diretório (dirs já estão `atlas-*`)
- **Dependências:** T01
- **Critério de done:** `grep -rn 'codex-\|cursor-\|claude-' packages/skills/atlas-*/SKILL.md` retorna zero matches relevantes a skill da cadeia
- **Validação local:**
  ```bash
  cd /Volumes/Dados/projetos/atlas-workflow && \
    grep -rn -E '(codex|cursor|claude)-(plan|sprint|task|slice|prd|direct)' packages/skills/ | wc -l
  # Esperado: 0
  ```

#### T03. Eliminar variante `*-orchestrated` em skills

- **Objetivo:** remover toda menção a `codex-plan-execute-orchestrated` (ou variantes claude/cursor) das skills. Comportamento orchestrated-per-slice torna-se default interno do executor.
- **Referência:** `packages/skills/atlas-plan-handoff/SKILL.md` linha 19 (cadeia documentada)
- **Pré-condições:** T02
- **Mudança esperada:** zero ocorrência de `plan-execute-orchestrated` em arquivos de skill/template/manifest; nota inline em `atlas-plan-execute/SKILL.md` explicando que orchestrated-per-slice é modo default selecionável via metadata do plano.
- **Invariantes preservados:** I2
- **Não mudar:** estrutura da state machine documentada
- **Dependências:** T02
- **Critério de done:** grep retorna 0 matches
- **Validação local:**
  ```bash
  cd /Volumes/Dados/projetos/atlas-workflow && \
    grep -rn 'plan-execute-orchestrated' packages/ hooks/ plugin-manifests/ | wc -l
  # Esperado: 0
  ```

#### T04. Substituir `§14` → `§8` em skills e templates

- **Objetivo:** referências `§14`, `(§14)`, "Section 8 ... (§14)" passam a `§8` simples, sem tag legada. Notas tipo "tag (§14) opcional — legado" são removidas.
- **Referência:** matches conhecidos em `atlas-plan-handoff/SKILL.md:108`, `atlas-slice-review/SKILL.md:17,30`, `atlas-task-validator/SKILL.md:19,55,65,101`, `atlas-plan-execute/SKILL.md:33,61`, `orchestrator/README.md:289`, `orchestrator/atlas_workflows_config.md:211`, `packages/templates/PLAN_TEMPLATE.md` nota inline.
- **Pré-condições:** T02, T03
- **Mudança esperada:** zero matches `§14` em arquivos distribuídos (exceto se houver menção em `archive/` que não tocamos).
- **Invariantes preservados:** I1 (limpeza)
- **Não mudar:** numeração de seções no template (já está em 8)
- **Dependências:** T02, T03
- **Critério de done:** grep zero, exceto archive/
- **Validação local:**
  ```bash
  cd /Volumes/Dados/projetos/atlas-workflow && \
    grep -rn '§14' packages/ hooks/ plugin-manifests/ README.md | wc -l
  # Esperado: 0
  ```

#### T05. Atualizar `plugin-manifests/*/plugin.json` (descrição + skills count)

- **Objetivo:** descrições deixam de mencionar "21 skills (claude/cursor/codex)" e passam a "7 skills atlas-* + orquestrador + 5 templates canônicos".
- **Pré-condições:** T02–T04
- **Mudança esperada:** `plugin-manifests/claude/plugin.json` e `plugin-manifests/codex/plugin.json` com descrição atualizada. Versão permanece `__VERSION__` (placeholder de release.yml).
- **Invariantes preservados:** I1
- **Não mudar:** `mcpServers`, `author`, `keywords`
- **Dependências:** T04
- **Critério de done:** `grep -c '21 skills' plugin-manifests/*/plugin.json` retorna 0
- **Validação local:**
  ```bash
  cd /Volumes/Dados/projetos/atlas-workflow && \
    cat plugin-manifests/claude/plugin.json plugin-manifests/codex/plugin.json | grep -E '(description|skills)'
  ```

### Slice B — Kill Family Dimension no MCP (ConfigKill)

#### T06. Remover tool `atlas_lock_family` do MCP server

- **Objetivo:** deletar handler, schema, registro no list_tools, validações helper (`validateFamilyConfig`, `assertDispatchFamily`).
- **Referência:** `packages/mcp-server/server.js` linhas 263 (registro), 897 (`validateFamilyConfig`), 1057 (handler `atlas_lock_family`), 1139 (`assertDispatchFamily`), 1530 (schema).
- **Pré-condições:** Slice A completa
- **Mudança esperada:** server.js sem nenhuma referência a `atlas_lock_family`; sem regex `^```yaml\n(claude|cursor|codex):` (linha 140); sem helper `families[family]`.
- **Invariantes preservados:** I1
- **Não mudar:** `atlas_ping`, `atlas_run_state`, `atlas_verify_artifact`, `atlas_scan_prd`, `atlas_verify_template_conformance`, `atlas_assert_after_plan`
- **Não fazer:** deletar `atlas_lock_dispatch` (apenas remover dimensão família dele em T08)
- **Dependências:** T05
- **Riscos:** quebra clientes v0.2.0 — documentado em CHANGELOG na Slice E
- **Critério de done:** node carrega server.js sem erro; `tools/list` não retorna `atlas_lock_family`
- **Validação local:**
  ```bash
  cd /Volumes/Dados/projetos/atlas-workflow && \
    node -e "require('./packages/mcp-server/server.js')" && \
    grep -n 'atlas_lock_family\|validateFamilyConfig\|assertDispatchFamily' packages/mcp-server/server.js | wc -l
  # Esperado: 0
  ```

#### T07. Remover param `family` de `atlas_preflight`

- **Objetivo:** schema do `atlas_preflight` deixa de aceitar `family`. Handler usa apenas `run_id`, `mode`, `expected_version`, `project_root`.
- **Referência:** `packages/mcp-server/server.js` linhas 921 (handler), 1519 (schema `required: ['run_id', 'family', 'mode']` → `['run_id', 'mode']`).
- **Pré-condições:** T06
- **Mudança esperada:** preflight passa sem `family`; lock de "routing" interno some ou vira lock só de versão/modo.
- **Invariantes preservados:** I1
- **Não mudar:** validação de `expected_version` (continua exigindo `0.3.0`)
- **Dependências:** T06
- **Critério de done:** chamar `atlas_preflight` sem `family` retorna `passed`; chamar com `family` retorna erro de schema (ou ignora silencioso — decisão durante implementação documentada em §6.1)
- **Validação local:**
  ```bash
  cd /Volumes/Dados/projetos/atlas-workflow && \
    node -e "const s=require('./packages/mcp-server/server.js'); console.log(JSON.stringify(s.tools?.find(t=>t.name==='atlas_preflight')||'check schema manually'))" 2>&1 | head -20
  ```

#### T08. Remover dimensão família de `atlas_lock_dispatch`

- **Objetivo:** `atlas_lock_dispatch` vira lock só de fase (`phase: prd|plan_handoff|plan_execute|slice_review`), sem cruzamento com família.
- **Referência:** `packages/mcp-server/server.js` linhas 1155 (`assertDispatchFamily` call), 1224 (`active: { phase, family: ... }`), 1535 (schema), 1551 (schema phase variant).
- **Pré-condições:** T07
- **Mudança esperada:** schema sem `family`; estado interno não persiste `family`; assertion `assertDispatchFamily` deletada (já removida em T06 mas verificar chamadas residuais).
- **Invariantes preservados:** I1
- **Não mudar:** estados de fase (`start`/`complete`), `validator_status`
- **Dependências:** T07
- **Critério de done:** `atlas_lock_dispatch({phase: 'plan_execute', action: 'start'})` retorna ok sem `family`
- **Validação local:**
  ```bash
  cd /Volumes/Dados/projetos/atlas-workflow && \
    grep -n "family" packages/mcp-server/server.js | grep -v -E "(comment|//)" | wc -l
  # Esperado: 0 (ou só strings em comments/docs)
  ```

#### T09. Deletar `packages/orchestrator/atlas_workflows_config.md`

- **Objetivo:** arquivo de config família removido integralmente. Conteúdo relevante (sequência full/direct/interview-only) migra para `packages/orchestrator/README.md` como doc estático, sem semântica de roteamento.
- **Pré-condições:** T06–T08
- **Mudança esperada:** arquivo deletado; `README.md` do orchestrator atualizado com seção "Sequências canônicas" baseada no conteúdo migrado, sem tabela `families:`.
- **Invariantes preservados:** I1
- **Não mudar:** outros arquivos em `packages/orchestrator/` (commands/, defaults/, references/, skills/) salvo se referenciam config explicitamente
- **Dependências:** T08
- **Critério de done:** `ls packages/orchestrator/atlas_workflows_config.md` retorna "No such file"; `README.md` do orchestrator contém seção "Sequências canônicas"
- **Validação local:**
  ```bash
  cd /Volumes/Dados/projetos/atlas-workflow && \
    ! test -f packages/orchestrator/atlas_workflows_config.md && \
    grep -q 'Sequências canônicas' packages/orchestrator/README.md && echo OK
  ```

#### T10. Smoke MCP pós-ConfigKill

- **Objetivo:** verificar que servidor inicializa, `atlas_ping` retorna `ok`, `tools/list` lista apenas tools esperadas.
- **Pré-condições:** T06–T09
- **Mudança esperada:** logs limpos; tools listadas: `atlas_ping`, `atlas_run_state`, `atlas_preflight`, `atlas_verify_artifact`, `atlas_scan_prd`, `atlas_verify_template_conformance`, `atlas_lock_dispatch`, `atlas_assert_after_plan` (8 tools — `atlas_lock_family` removida).
- **Dependências:** T09
- **Critério de done:** smoke verde
- **Validação local:**
  ```bash
  cd /Volumes/Dados/projetos/atlas-workflow && \
    node packages/mcp-server/server.js < /dev/null &
    SERVER_PID=$!; sleep 1; kill $SERVER_PID 2>/dev/null
    # Para teste real de tools/list usar cliente MCP — aqui só boot.
  ```

### Slice C — Validator Subagent + State File + Veredito JSON (D2 + S1 + S8)

#### T11. Definir schema state file `.atlas/state/<run_id>/<slice>.json`

- **Objetivo:** schema canônico do boundary executor→validator documentado em `packages/templates/STATE_FILE_SCHEMA.md` (novo arquivo de template). Apenas para o schema; nada de instrução procedural — skills consomem.
- **Mudança esperada:** novo arquivo `packages/templates/STATE_FILE_SCHEMA.md` com schema JSON inline:
  ```json
  {
    "run_id": "string (uuid ou slug)",
    "slice": "string (ex.: 'A', 'B', 'C')",
    "tasks": ["T01", "T02"],
    "files_changed": ["packages/foo.js"],
    "diff_stat": "N files, +X -Y",
    "plan_path": ".atlas/plans/<id>.plan.md",
    "boundary_refs": ["§2.I3", "§5.T11"],
    "executed_at": "ISO8601",
    "executor_skill": "atlas-plan-execute"
  }
  ```
- **Invariantes preservados:** I4
- **Não mudar:** schemas existentes em `PLAN_TEMPLATE.md` / `PRD_TEMPLATE.md`
- **Dependências:** Slice B completa
- **Critério de done:** arquivo existe; bloco JSON parseável
- **Validação local:**
  ```bash
  cd /Volumes/Dados/projetos/atlas-workflow && \
    test -f packages/templates/STATE_FILE_SCHEMA.md && \
    grep -q 'run_id' packages/templates/STATE_FILE_SCHEMA.md && echo OK
  ```

#### T12. Atualizar `atlas-plan-execute/SKILL.md` para gravar state file

- **Objetivo:** skill instrui executor a, ao entrar em `slice_validating`, gravar `.atlas/state/<run_id>/<slice>.json` conforme schema T11 antes de invocar validator subagent.
- **Pré-condições:** T11
- **Mudança esperada:** SKILL.md tem nova subseção "State file boundary" com pseudo-código de gravação e referência a `STATE_FILE_SCHEMA.md`.
- **Invariantes preservados:** I4, I7 (state file é projeção; SSoT real continua em `atlas_run_state` MCP em T14)
- **Não mudar:** state machine principal
- **Dependências:** T11
- **Critério de done:** SKILL.md contém referência a `STATE_FILE_SCHEMA.md` e bloco "State file boundary"

#### T13. Registrar `atlas-task-validator` como `subagent_type`

- **Objetivo:** validator deixa de ser apenas skill — vira subagent_type real, invocável via `Agent(subagent_type: "atlas-task-validator", prompt: <state_path>)`.
- **Mudança esperada:**
  - Criar/atualizar `packages/skills/atlas-task-validator/SKILL.md` com frontmatter de subagent (campo `subagent_type: true` ou equivalente conforme convenção Claude Code Plugin SDK — investigar durante T13).
  - Validator lê apenas `state_path` como input; nunca aceita prompt inline com contrato.
  - Documentar em `atlas-plan-execute/SKILL.md` a chamada correta.
- **Pré-condições:** T11, T12
- **Invariantes preservados:** I3
- **Não mudar:** lógica interna do validator (continua cold review)
- **Dependências:** T12
- **Riscos:** Plugin SDK pode não suportar registro de subagent via SKILL.md — investigar; fallback é manifest do plugin declarar `subagents:` separado.
- **Critério de done:** chamada `Agent(subagent_type: "atlas-task-validator", ...)` em teste manual não erra com `Agent type not found`.
- **Validação local:**
  ```bash
  cd /Volumes/Dados/projetos/atlas-workflow && \
    grep -n 'subagent_type\|subagents' plugin-manifests/claude/plugin.json packages/skills/atlas-task-validator/SKILL.md
  ```

#### T14. Veredito JSON estruturado no `atlas-task-validator`

- **Objetivo:** SKILL.md instrui validator a emitir JSON estrito como output final, antes de qualquer prosa explicativa.
- **Mudança esperada:** seção "Output contract" no SKILL.md com schema:
  ```json
  {
    "verdict": "pass | fail | pass_with_observations",
    "findings": [{"severity": "P1|P2|P3", "file": "string", "line": 0, "msg": "string"}],
    "observations": [{"file": "string", "line": 0, "msg": "string"}],
    "boundary_violations": [{"file": "string", "reason": "string"}]
  }
  ```
- **Pré-condições:** T13
- **Invariantes preservados:** I6
- **Não mudar:** critérios de severidade P1/P2/P3 já documentados
- **Dependências:** T13
- **Critério de done:** SKILL.md tem seção "Output contract" com bloco JSON exemplo
- **Validação local:**
  ```bash
  cd /Volumes/Dados/projetos/atlas-workflow && \
    grep -A2 'Output contract' packages/skills/atlas-task-validator/SKILL.md | head -10
  ```

#### T15. Atualizar executor para parse JSON e decisão por campo

- **Objetivo:** `atlas-plan-execute/SKILL.md` instrui que, após retorno do validator subagent, faz `JSON.parse(output)`, lê `verdict`, e decide:
  - `pass` → `slice_done`
  - `pass_with_observations` → `slice_done` + log de observations
  - `fail` → `repairing` (máx 2 ciclos) ou `blocked`
- **Pré-condições:** T14
- **Invariantes preservados:** I6
- **Não mudar:** limite de 2 repairs
- **Dependências:** T14
- **Critério de done:** SKILL.md contém pseudo-código de parse JSON e decisão
- **Validação local:**
  ```bash
  cd /Volumes/Dados/projetos/atlas-workflow && \
    grep -A3 'JSON.parse\|verdict' packages/skills/atlas-plan-execute/SKILL.md | head -15
  ```

### Slice D — SSoT MCP + Paths + Espelhamento Todo + Flag --review (S7 + S2 + S6' + S3)

#### T16. Promover `atlas_run_state` MCP a SSoT de estado nas skills

- **Objetivo:** todas as skills que leem/escrevem estado de run (handoff, execute, validator, slice-review) passam a usar `atlas_run_state` tool em vez de file IO direto em `.atlas-run/`. File IO mantido apenas como fallback explícito com warning.
- **Referência:** `packages/mcp-server/server.js` linha 258 (tool já registrada), linha 536 (handler `get`), linha 542+ (handler `set`).
- **Pré-condições:** Slice C completa
- **Mudança esperada:** 4 SKILL.md (handoff, execute, validator, slice-review) com seção "State persistence" referenciando `atlas_run_state` como primary, com nota de fallback.
- **Invariantes preservados:** I7
- **Não mudar:** schema interno do `atlas_run_state` (já existe)
- **Dependências:** Slice C
- **Critério de done:** grep `atlas_run_state` em 4 SKILL.md retorna ≥1 por arquivo
- **Validação local:**
  ```bash
  cd /Volumes/Dados/projetos/atlas-workflow && \
    for f in atlas-plan-handoff atlas-plan-execute atlas-task-validator atlas-slice-review; do
      grep -c 'atlas_run_state' "packages/skills/$f/SKILL.md" || echo "0 $f"
    done
  ```

#### T17. Paths canônicos `.atlas/plans/` + `.atlas/state/` + migration order

- **Objetivo:** skill `atlas-plan-handoff` grava em `.atlas/plans/`. Leitura aceita ordem: `.atlas/plans/` → `.cursor/plans/` → `.codex/plans/` com deprecation warning nos 2 últimos. Skill `atlas-plan-execute` lê plano no mesmo order.
- **Pré-condições:** T16
- **Mudança esperada:** 2 SKILL.md (handoff, execute) com seção "Plan path resolution" explícita; `STATE_FILE_SCHEMA.md` consistente com `.atlas/state/<run_id>/<slice>.json`.
- **Invariantes preservados:** I5
- **Não mudar:** schema interno do plano
- **Dependências:** T16
- **Critério de done:** SKILL.md mencionam explicitamente os 3 paths e deprecation
- **Validação local:**
  ```bash
  cd /Volumes/Dados/projetos/atlas-workflow && \
    grep -l '.atlas/plans/' packages/skills/atlas-plan-handoff/SKILL.md packages/skills/atlas-plan-execute/SKILL.md && \
    grep -l 'deprecat' packages/skills/atlas-plan-handoff/SKILL.md
  ```

#### T18. Espelhamento declarativo plano→todo nativo no executor

- **Objetivo:** `atlas-plan-execute/SKILL.md` instrui que ao entrar em `implementing` (primeira vez por slice), executor espelha tasks do plano no todo nativo do cliente. Plano = SSoT.
- **Pré-condições:** T17
- **Mudança esperada:** nova subseção "Native todo mirror" em `atlas-plan-execute/SKILL.md` com:
  - Ferramenta por cliente: TodoWrite (Claude Code), todo_write (Cursor), tasks (Codex App), genérico para outros.
  - Estados: `pending`→`in_progress`→`completed` mapeados pra state machine.
  - Regra: divergência ⇒ sincronizar do plano pro todo, nunca o inverso.
  - Anti-padrão: criar todos paralelos não derivados do plano.
- **Invariantes preservados:** I8
- **Não mudar:** state machine principal
- **Dependências:** T17
- **Critério de done:** seção "Native todo mirror" presente
- **Validação local:**
  ```bash
  cd /Volumes/Dados/projetos/atlas-workflow && \
    grep -A5 'Native todo mirror' packages/skills/atlas-plan-execute/SKILL.md | head -10
  ```

#### T19. Gate `--review` explícito para `atlas-slice-review`

- **Objetivo:** `atlas-slice-review/SKILL.md` declara que skill só é invocada se flag `--review` presente. `atlas-plan-execute/SKILL.md` declara que ao fechar `slice_done`, verifica flag `--review`; se ausente, encerra; se presente, despacha `atlas-slice-review`.
- **Pré-condições:** T18
- **Mudança esperada:** 2 SKILL.md com gate explícito; remover qualquer auto-trigger heurístico anteriormente sugerido.
- **Invariantes preservados:** I9
- **Não mudar:** lógica interna do slice-review
- **Dependências:** T18
- **Critério de done:** ambos SKILL.md mencionam `--review` como única condição de disparo
- **Validação local:**
  ```bash
  cd /Volumes/Dados/projetos/atlas-workflow && \
    grep -n '\-\-review' packages/skills/atlas-slice-review/SKILL.md packages/skills/atlas-plan-execute/SKILL.md
  ```

#### T20. Atualizar hook `hooks/claude/atlas-workflow-hook.js` (se necessário)

- **Objetivo:** se hook referencia famílias ou paths antigos, atualizar para neutro/canônico.
- **Pré-condições:** T19
- **Mudança esperada:** ler arquivo, identificar refs, ajustar. Se hook não referencia, marcar como no-op.
- **Invariantes preservados:** I1, I5
- **Não mudar:** comportamento de transparência (0-token overhead)
- **Dependências:** T19
- **Critério de done:** hook sem refs `codex-/cursor-/claude-` específicos da cadeia ou `.cursor/plans/` / `.codex/plans/` como único path
- **Validação local:**
  ```bash
  cd /Volumes/Dados/projetos/atlas-workflow && \
    grep -nE '(codex|cursor|claude)-(plan|sprint|task|slice|prd|direct)|\.codex/plans|\.cursor/plans' hooks/claude/atlas-workflow-hook.js | wc -l
  # Esperado: 0
  ```

### Slice E — README + Manifest + Validação Final (S5 + bump versão)

#### T21. Reescrever `README.md` do plugin com seção "Como funciona"

- **Objetivo:** raiz `/Volumes/Dados/projetos/atlas-workflow/README.md` ganha seção "Como funciona" com:
  - Tabela skill→input→output→próxima skill (7 linhas).
  - Diagrama ASCII da state machine: `ready → implementing → gating → repairing → task_done → slice_validating → slice_done | blocked`.
  - Nota: "Atlas é família única. Cliente (Claude Code, Cursor, Codex App) é executor das skills, não família. Não há mais roteamento por família."
- **Pré-condições:** Slice D completa
- **Mudança esperada:** README com nova seção, ~80–120 linhas adicionadas; outras seções intactas.
- **Invariantes preservados:** I1
- **Não mudar:** CHANGELOG.md (atualizado em T22), PATCH_PROCEDURE.md, raycast/, archive/
- **Dependências:** Slice D
- **Critério de done:** README contém heading "## Como funciona" e tabela + diagrama
- **Validação local:**
  ```bash
  cd /Volumes/Dados/projetos/atlas-workflow && \
    grep -q '## Como funciona' README.md && \
    grep -q 'ready → implementing' README.md && echo OK
  ```

#### T22. Bump VERSION + entrada CHANGELOG.md v0.3.0

- **Objetivo:** `VERSION` passa a `0.3.0`. `CHANGELOG.md` ganha entrada `## v0.3.0 — Família única + validator subagent` com bullets de breaking changes e novas features.
- **Pré-condições:** T21
- **Mudança esperada:** 2 arquivos editados.
- **Invariantes preservados:** I1, I11 (apenas edição local, sem commit/tag)
- **Não mudar:** entradas anteriores do CHANGELOG.md
- **Dependências:** T21
- **Critério de done:** `cat VERSION` retorna `0.3.0`; CHANGELOG tem header `## v0.3.0`
- **Validação local:**
  ```bash
  cd /Volumes/Dados/projetos/atlas-workflow && \
    test "$(cat VERSION)" = "0.3.0" && \
    grep -q '## v0.3.0\|## \[0.3.0\]' CHANGELOG.md && echo OK
  ```

#### T23. Validação final

- **Objetivo:** smoke ponta-a-ponta: MCP boot, `atlas_ping` retorna `version: 0.3.0`, zero refs legadas em arquivos distribuídos.
- **Pré-condições:** T01–T22
- **Mudança esperada:** nenhuma — apenas validação.
- **Dependências:** T22
- **Critério de done:** todos os greps abaixo retornam 0; node carrega server.js sem erro
- **Validação local:**
  ```bash
  cd /Volumes/Dados/projetos/atlas-workflow && \
    node -e "require('./packages/mcp-server/server.js')" && \
    echo "=== Skill name refs ==="; \
    grep -rn -E '(codex|cursor|claude)-(plan|sprint|task|slice|prd|direct)' packages/skills/ packages/orchestrator/ packages/templates/ hooks/ plugin-manifests/ | grep -v 'archive/' | wc -l && \
    echo "=== §14 refs ==="; \
    grep -rn '§14' packages/ hooks/ plugin-manifests/ README.md | wc -l && \
    echo "=== orchestrated variant ==="; \
    grep -rn 'plan-execute-orchestrated' packages/ hooks/ plugin-manifests/ | wc -l && \
    echo "=== atlas_lock_family residual ==="; \
    grep -rn 'atlas_lock_family\|validateFamilyConfig\|assertDispatchFamily' packages/ | wc -l && \
    echo "=== family param ==="; \
    grep -n 'family' packages/mcp-server/server.js | grep -v -E "(//|comment)" | wc -l && \
    echo "=== VERSION ==="; cat VERSION
  # Esperado: todos os wc -l retornam 0; VERSION = 0.3.0
  ```
- **Verificação manual (recomendada):**
  1. Em sessão separada do Claude Code, instalar plugin local: simular `claude plugins install` com worktree atual.
  2. Rodar `Agent(subagent_type: "atlas-task-validator", prompt: ".atlas/state/<run_id>/A.json")` em projeto fictício — não deve retornar `Agent type not found`.
  3. Inspecionar `tools/list` MCP — confirmar 8 tools sem `atlas_lock_family`.

---

## 6. Contratos técnicos

### 6.1 Comportamento ao receber `family` legado

| Tool | v0.2.0 | v0.3.0 (esta entrega) |
|------|--------|-----------------------|
| `atlas_preflight({family: "claude", ...})` | aceita, valida | **rejeita** com erro `unknown_property: family` (preferir hard fail vs ignore silencioso — facilita debug de clientes desatualizados) |
| `atlas_lock_dispatch({family, phase, action})` | aceita | **rejeita** com mesmo erro |
| `atlas_lock_family(...)` | aceita | **rejeita** com `method_not_found` |

Justificativa: hard fail força clientes v0.2.0 a atualizar — silencioso esconde regressão.

### 6.2 Schema state file `.atlas/state/<run_id>/<slice>.json`

| Campo | Tipo | Obrigatório | Descrição |
|-------|------|-------------|-----------|
| `run_id` | string (slug/uuid) | sim | identificador único da run |
| `slice` | string | sim | letra/ID da slice (`A`, `B`...) |
| `tasks` | array<string> | sim | IDs das tasks executadas nesta slice |
| `files_changed` | array<string> | sim | paths relativos modificados |
| `diff_stat` | string | sim | output de `git diff --stat` ou equivalente |
| `plan_path` | string | sim | path do plano relativo ao repo |
| `boundary_refs` | array<string> | sim | refs `§N.IK` ou `§N.TKK` do plano |
| `executed_at` | string (ISO8601) | sim | timestamp de gravação |
| `executor_skill` | string | sim | nome da skill executora (sempre `atlas-plan-execute` pós-refactor) |

### 6.3 Schema veredito validator

| Campo | Tipo | Obrigatório | Descrição |
|-------|------|-------------|-----------|
| `verdict` | enum (`pass`/`fail`/`pass_with_observations`) | sim | decisão final |
| `findings` | array | sim | issues bloqueantes ou alertas; vazio se `pass` |
| `findings[].severity` | enum (`P1`/`P2`/`P3`) | sim | criticidade |
| `findings[].file` | string | sim | path relativo |
| `findings[].line` | number | sim | linha (0 se não aplicável) |
| `findings[].msg` | string | sim | descrição |
| `observations` | array | não | itens fora de boundary mas notáveis |
| `boundary_violations` | array | não | arquivos modificados fora do boundary declarado no state file |

Decisão do executor:
- `verdict == "pass"` → `slice_done`
- `verdict == "pass_with_observations"` → `slice_done` + log
- `verdict == "fail"` → `repairing` (≤2 ciclos) → se ainda fail, `blocked`

---

## 7. Slices

| Slice | Tasks | Objetivo | Boundary de diff esperado |
|-------|-------|----------|---------------------------|
| A | T01–T05 | Naming + legado: rename `codex-*` no conteúdo, eliminar `*-orchestrated`, `§14`→`§8`, atualizar manifest | `packages/skills/atlas-*/SKILL.md`, `packages/templates/*.md`, `packages/orchestrator/README.md`, `plugin-manifests/*/plugin.json`, `.atlas/state/<run_id>/A-audit.txt` |
| B | T06–T10 | Kill family no MCP: remover `atlas_lock_family`, param `family` em preflight/dispatch, deletar `atlas_workflows_config.md` | `packages/mcp-server/server.js`, `packages/orchestrator/atlas_workflows_config.md` (delete), `packages/orchestrator/README.md` |
| C | T11–T15 | Validator subagent + state file + veredito JSON | `packages/templates/STATE_FILE_SCHEMA.md` (novo), `packages/skills/atlas-plan-execute/SKILL.md`, `packages/skills/atlas-task-validator/SKILL.md`, `plugin-manifests/claude/plugin.json` (registro subagent se necessário) |
| D | T16–T20 | `atlas_run_state` SSoT, paths `.atlas/`, espelhamento todo, gate `--review` | `packages/skills/atlas-plan-handoff/SKILL.md`, `packages/skills/atlas-plan-execute/SKILL.md`, `packages/skills/atlas-slice-review/SKILL.md`, `packages/skills/atlas-task-validator/SKILL.md`, `hooks/claude/atlas-workflow-hook.js` |
| E | T21–T23 | README, bump versão, CHANGELOG, smoke final | `README.md`, `VERSION`, `CHANGELOG.md` |

Ordem: **A → B → C → D → E**. Validator subagent: boundary do diff por slice + invariantes §2.

---

## 8. Validação e checklist (validator)

Referência §2 (invariantes I1–I11) + §6 (contratos).

```bash
cd /Volumes/Dados/projetos/atlas-workflow && \
  node -e "require('./packages/mcp-server/server.js')" && \
  cat VERSION
```

- [ ] I1: zero matches `(codex|cursor|claude)-(plan|sprint|task|slice|prd|direct)` em `packages/`, `hooks/`, `plugin-manifests/`, `README.md`
- [ ] I2: zero matches `plan-execute-orchestrated`
- [ ] I3: `packages/skills/atlas-task-validator/SKILL.md` declara registro como subagent_type; `atlas-plan-execute/SKILL.md` chama via `Agent(subagent_type: ...)`
- [ ] I4: `packages/templates/STATE_FILE_SCHEMA.md` existe com schema completo
- [ ] I5: `atlas-plan-handoff/SKILL.md` documenta migration order `.atlas/` → `.cursor/` → `.codex/`
- [ ] I6: `atlas-task-validator/SKILL.md` tem seção "Output contract" com JSON schema; `atlas-plan-execute/SKILL.md` instrui parse JSON e decisão por campo
- [ ] I7: 4 SKILL.md (handoff, execute, validator, slice-review) referenciam `atlas_run_state` como primary
- [ ] I8: `atlas-plan-execute/SKILL.md` tem seção "Native todo mirror" com mapeamento por cliente
- [ ] I9: `atlas-slice-review/SKILL.md` declara `--review` como única condição de disparo
- [ ] I10: smoke `atlas_ping` retorna `version: 0.3.0` (validação manual após Slice E)
- [ ] I11: nenhum `git commit`/`git push`/`git tag` foi executado durante a run; `archive/v0.1.10/` e `raycast/` intactos
- [ ] §4 limpeza: zero matches `§14` em `packages/`, `hooks/`, `plugin-manifests/`, `README.md`
- [ ] `VERSION` = `0.3.0`
- [ ] `CHANGELOG.md` tem entrada `## v0.3.0` com breaking changes documentadas
- [ ] `plugin-manifests/*/plugin.json` descrição não menciona "21 skills"

---

## 9. Perguntas em aberto e bloqueios reais

- **B1 (informativo, não bloqueante):** `Agent type` registration via SKILL.md vs plugin manifest. Em T13, se SDK do Claude Code Plugin não suportar registro de subagent via SKILL.md, declarar em `plugin-manifests/claude/plugin.json` como `subagents: [{name: "atlas-task-validator", ...}]` (formato exato a confirmar lendo SDK doc durante T13). Fallback documentado, não bloqueia.
- **B2 (informativo):** comportamento de hard fail em T07 (rejeitar `family` legado) vs ignore silencioso — plano adota hard fail; se durante execução surgir cliente legado conhecido, decisão pode reverter para ignore com warning. Reavaliar em T07 se necessário; mudança de decisão exige patch no plano.

> Sem bloqueios ativos. Executor pode iniciar Slice A.
