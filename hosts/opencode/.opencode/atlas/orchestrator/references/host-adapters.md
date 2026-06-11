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
| env `ATLAS_HOST=opencode` (injetado por `opencode.json`) | `opencode` |
| env `ATLAS_HOST=pi` (injetado pela config do `pi-mcp-adapter`) | `pi` |
| nenhum | `generic` |

## Matriz de adapters

| Concern | `claude` (Claude Code) | `codex` (Codex App) | `opencode` | `pi` (pi cli) | `generic` |
|---------|------------------------|---------------------|------------|---------------|-----------|
| Disparo de subagente | `Agent(subagent_type: "<name>", prompt: "<state_path>")` | `spawn_agent(agent_type: "<name>", items: [{ type: "text", text: "<state_path>" }])` | `@<name>` (ou auto) com `<state_path>` | tool `subagent({ agent: "<name>", task: "<state_path>", context: "fresh" })` (pi-subagents) | subagente nativo do host, passando só `<state_path>` |
| Registro do subagente | `agents/<name>.md` na raiz do plugin | `.codex/agents/<name>.toml` (custom agent nativo; `developer_instructions` carrega o `SKILL.md`) | `.opencode/agents/<name>.md` (`mode: subagent`) | `.pi/agents/<name>.md` (pi-subagents; frontmatter `name`+`description`+`tools`; **`SKILL.md` canônico embutido no corpo** porque o pi não tem skill loader no sub-agente — fonte única segue `packages/skills/<name>/SKILL.md`, agente é cópia gerada por `build/gen-host-agent.mjs`) | mecanismo nativo equivalente |
| Topologia do validador frio (G4) | **`sibling`** | **`sibling`** | **`sibling`** | **`sibling`** | **`sibling`** |
| Join síncrono (gate JOIN) | `self_evident` (`Agent()` bloqueante) | `self_evident` (confirmado em produção) | `self_evident` (`@<name>` bloqueante) | `must_report` (depende de `pi-subagents`; hard-fail sem report) | `must_report` (indeterminado; hard-fail sem report) |
| Todo nativo | `TodoWrite` | `tasks` | `todowrite` | nenhum (segue sem mirror) | nenhum (segue sem mirror) |
| Config MCP | `plugin.json` `mcpServers` | `.mcp.json` | `opencode.json` `mcp.<name>` (`type:"local"`, `environment.ATLAS_HOST=opencode`) | `.mcp.json` no root (`pi-mcp-adapter`; `env.ATLAS_HOST=pi`); tools chegam proxiadas/prefixadas `atlas_workflow_<tool>` | host MCP-capaz |
| Deps externas obrigatórias | — | — | — | **`pi-mcp-adapter` + `pi-subagents`** (DEC-005) | — |
| Estado de run | `atlas_run_state` (MCP) | `atlas_run_state` (MCP) | `atlas_run_state` (MCP) | `atlas_run_state` (MCP) | `atlas_run_state` (MCP) |
| Escrita de plano | `.atlas/plans/` | `.atlas/plans/` | `.atlas/plans/` | `.atlas/plans/` | `.atlas/plans/` |
| Leitura de plano (ordem) | `.atlas/plans/` → `.cursor/plans/` → `.codex/plans/` | idem | idem | idem | idem |

`.cursor/plans/` e `.codex/plans/` são lidos com deprecation warning por 1 release; escrita só em `.atlas/plans/`. **opencode** instala via `.opencode/` + `opencode.json` (`hosts/opencode/`). **pi** instala via `mcp.json` + `agents/` + `skills/` (`hosts/pi/`) e exige as 2 deps obrigatórias; sem qualquer uma o preflight aborta (gate PREREQ).

## Contrato `atlas_capabilities` (schema v2)

Campos retornados (DEC-007):

| Campo | Tipo | Significado |
|---|---|---|
| `host` / `host_label` / `detected_via` | string | host detectado e como |
| `schema_version` | int | versão do contrato (atual: **3**) |
| `subagent_dispatch` | obj | `{mechanism, example, registration}` — verbo nativo de dispatch |
| `validator_dispatch` | obj | `{dispatcher: 'orchestrator', join: {sync, confidence, mechanism}}` — topologia é sempre sibling; `join` declara a capability de join síncrono usada pelo gate JOIN |
| `todo_tool` | string\|null | tool de todo nativa; `null` = seguir sem mirror (não-essencial) |
| `hooks` | obj | `{supported, mechanism}` — suporte a hooks pré/pós tool |
| `capabilities_flags` | obj | `{subagent_available, mcp_available, todo_available}` |
| `prerequisites` | obj | `{essential:[…], non_essential:[…]}` — quais flags são hard-fail |
| `plan_paths` / `state_backend` / `state_dir` | — | **portáveis** (iguais em todo host) |
| `known_hosts` | string[] | hosts registrados em `HOST_ADAPTERS` |

### Política de versionamento (`schema_version`)

- **Aditivo** (campo novo opcional) → mantém compat; consumidores **devem ignorar campos desconhecidos**. (v1→v2 foi aditivo: `capabilities_flags`, `hooks`, `prerequisites`.)
- **Remoção/renomeação/mudança de semântica** → bump + nota de migração + revisão das skills consumidoras.

### Pré-requisitos de determinismo (DEC-004)

`prerequisites.essential` (`subagent_available`, `mcp_available`) são **hard-fail**: host sem qualquer um é rejeitado no preflight, qualquer tamanho de tarefa, sem degradação/inline. `prerequisites.non_essential` (`todo_available`) apenas segue sem o recurso, registrando. O executor consome esse contrato no preflight (S09).

**Gate `PREREQ` no `atlas_preflight`:** é a **primeira** verificação (precede versão/lock/modo). Mescla as flags do perfil do host com a disponibilidade real reportada em `host_capabilities` (override). Ex.: pi sem `pi-mcp-adapter`/`pi-subagents` → o adapter reporta `{"subagent_available":false}` → `status:"blocked"`, `gate:"PREREQ"`, `missing_prerequisites:[…]`, `next_action` acionável. Host qualificado passa direto para o gate G10. Nunca há fallback inline.

### Transporte (S05 — spike, DEC-006)

**stdio único.** Confirmado pelo survey S01: opencode usa `type:"local"` (stdio) e o `pi-mcp-adapter` suporta stdio (com fallback HTTP interno do próprio adapter, transparente ao Atlas). Nenhum host-alvo (claude/codex/cursor/opencode/pi/generic) exige HTTP/SSE no MCP do Atlas. Não há abstração de transporte (YAGNI). Ponto de extensão: se um host futuro exigir HTTP/SSE, o boot fica isolado em `startStdioLoop()` (`server.js`) — trocar/adicionar transporte é localizado, sem tocar a lógica de tools/gates.

### Fronteira portável vs host-específico

- **Portável (vive no MCP, igual a todo host):** `plan_paths`, `state_backend`, `state_dir`, gates G1–G11, schema de state. Nunca depende de host.
- **Host-específico (vive em `HOST_ADAPTERS` + packaging):** `subagent_dispatch`, `todo_tool`, `hooks`, `capabilities_flags`. Variação resolvida por dado, não por ramo de código.

## Como uma skill consome

1. Chamar `atlas_capabilities` (sem args para autodetecção, ou `{host}` para forçar).
2. Ler `subagent_dispatch.mechanism` / `.example`, `todo_tool`, `plan_paths`.
3. Executar o verbo nativo correspondente. Nunca hardcodar o nome do host na prosa da skill. No Codex, `$<skill>` é ativação de skill in-context; execução/review usa custom agent nativo via `spawn_agent`.
4. Se `todo_tool` for `null`, seguir sem mirror de todo (não inventar tool).

## Adicionar um host novo

1. Adicionar entrada em `HOST_ADAPTERS` (`packages/mcp-server/server.js`).
2. Adicionar regra de detecção em `detectHost` se houver env próprio.
3. Adicionar linha na matriz de adapters.
4. Registrar o subagente no formato nativo do host (ex.: `agents/<name>.md` ou equivalente).

Sem tocar nas skills — elas já consomem o descritor.

## Perfil `generic` (DEC-004)

`generic` é o fallback para qualquer host MCP-capaz **com subagente nativo**. Não tem packaging próprio (não há bundle `generic`): o host usa seu mecanismo nativo de subagente + a config MCP do próprio host. O perfil **exige** subagente + MCP — `capabilities_flags {subagent:true, mcp:true}`. Host MCP-only **sem** subagente nativo fica **fora de escopo**: reportando `subagent_available:false` no preflight, o gate PREREQ aborta (não há degradação nem cold-review inline). Determinismo > alcance.

## Status multi-host

Todos os hosts-alvo do survey S01 estão implementados na matriz acima: `claude`, `codex`, `cursor` (carona no manifest claude), `opencode` (S06), `pi` (S07) e `generic`. Nenhum exige HTTP/SSE → stdio único (DEC-006/S05). Survey completo + fontes: `PRD_S01_host_survey.md`.
