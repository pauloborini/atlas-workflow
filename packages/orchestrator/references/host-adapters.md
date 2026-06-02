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
3. Adicionar linha nesta matriz.
4. Registrar o subagente no formato nativo do host (ex.: `agents/<name>.md` ou equivalente).

Sem tocar nas skills — elas já consomem o descritor.
