# Adapters de host

Fonte canônica do conhecimento host-específico do Atlas. As skills são host-agnósticas: em runtime devem consultar a tool MCP `atlas_capabilities` e usar o descritor retornado. Este documento é a referência estática equivalente (e o que o `atlas_capabilities` materializa em código no MCP server).

## Por que existe

Tools nativas do cliente (`Agent()`, `TodoWrite`, `tasks`, `$skill`) vivem no host, não no MCP. O servidor não consegue chamá-las nem fazer proxy — só **descrever** qual usar. Por isso o adapter é descritivo: centraliza o "qual verbo usar por host" num único lugar (código + este doc), eliminando prosa duplicada espalhada pelas skills.

## Fonte de verdade

1. **Runtime:** `atlas_capabilities` (MCP) — detecta host por env e retorna o descritor. Preferir sempre.
2. **Estático:** esta tabela — fallback de leitura/documentação quando o MCP não está disponível.

Os dois devem permanecer consistentes. O descritor em código vive em `packages/mcp-server/server.js` (`HOST_ADAPTERS`).

## Detecção de host

| Sinal | Host |
|-------|------|
| arg `host` explícito na chamada | o valor passado |
| env `ATLAS_HOST` | o valor da env |
| env `CLAUDE_PLUGIN_ROOT` presente | `claude` |
| env `CODEX_HOME` / `CODEX_PLUGIN_ROOT` | `codex` |
| nenhum | `generic` |

## Matriz de adapters

| Concern | `claude` (Claude Code) | `codex` (Codex App) | `generic` |
|---------|------------------------|---------------------|-----------|
| Disparo de subagente | `Agent(subagent_type: "<name>", prompt: "<state_path>")` | invocar `$<skill-name>` com `<state_path>` | subagente nativo do host, passando só `<state_path>` |
| Registro do subagente | `agents/<name>.md` na raiz do plugin | `agents/openai.yaml` por skill (`allow_implicit_invocation`) | mecanismo nativo equivalente |
| Todo nativo | `TodoWrite` | `tasks` | nenhum (degradar sem mirror) |
| Estado de run | `atlas_run_state` (MCP) | `atlas_run_state` (MCP) | `atlas_run_state` (MCP) |
| Escrita de plano | `.atlas/plans/` | `.atlas/plans/` | `.atlas/plans/` |
| Leitura de plano (ordem) | `.atlas/plans/` → `.cursor/plans/` → `.codex/plans/` | idem | idem |

`.cursor/plans/` e `.codex/plans/` são lidos com deprecation warning por 1 release; escrita só em `.atlas/plans/`.

## Como uma skill consome

1. Chamar `atlas_capabilities` (sem args para autodetecção, ou `{host}` para forçar).
2. Ler `subagent_dispatch.mechanism` / `.example`, `todo_tool`, `plan_paths`.
3. Executar o verbo nativo correspondente. Nunca hardcodar o nome do host na prosa da skill.
4. Se `todo_tool` for `null`, seguir sem mirror de todo (não inventar tool).

## Adicionar um host novo

1. Adicionar entrada em `HOST_ADAPTERS` (`packages/mcp-server/server.js`).
2. Adicionar regra de detecção em `detectHost` se houver env próprio.
3. Adicionar linha na matriz de adapters.
4. Registrar o subagente no formato nativo do host (ex.: `agents/<name>.md` ou equivalente).

Sem tocar nas skills — elas já consomem o descritor.

## Hosts-alvo (roadmap multi-host — S01 survey)

Hosts em expansão (`feature/multihost-expansion`). Esta seção é **design input** para os adapters (S06/S07); só vira "Matriz de adapters" acima quando implementado e sincronizado com `HOST_ADAPTERS`. Detalhe completo + fontes: `PRD_S01_host_survey.md`.

| Concern | `opencode` | `pi` (pi cli) |
|---|---|---|
| Disparo de subagente | nativo: `@<name>` ou auto por `description` | **`pi-subagents`** (extensão npm, obrigatória) |
| Registro do subagente | `.opencode/agents/<name>.md` (frontmatter `description`, `mode: subagent`) | manifesto do package + frontmatter (`mcp:server-name` p/ tools) |
| Skills | `.opencode/skills/` (`skills_use(name)`) | manifesto do package (Skills/Extensions) |
| Config MCP (stdio) | `opencode.json` → `mcp.<name> = {type:"local", command:[…], enabled, environment}` | **`pi-mcp-adapter`** (extensão npm, obrigatória) → `mcp.json` |
| Detecção | `ATLAS_HOST=opencode` explícito + presença de `.opencode/`/`opencode.json` (sem env distintivo garantido no subprocesso MCP) | `ATLAS_HOST=pi` explícito |
| Deps externas obrigatórias | nenhuma (nativo compatível) | **`pi-mcp-adapter` + `pi-subagents`** (DEC-005); ausentes → preflight aborta (DEC-004) |
| Transporte | stdio (`type:"local"`) | stdio (suportado pelo adapter) |

**Conclusão do survey:** opencode é compatível nativamente (sem dep); pi exige 2 add-ons obrigatórios. Nenhum host-alvo exige HTTP/SSE → stdio único confirmado (DEC-006, S05 vira spike). Detecção por env não é garantida em opencode/pi → registry prioriza `ATLAS_HOST` explícito (tratado em S04).
