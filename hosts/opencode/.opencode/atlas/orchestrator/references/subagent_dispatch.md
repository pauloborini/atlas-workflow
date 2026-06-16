# Despacho de sub-agent

Contrato host-agnóstico para Cursor, Codex e Claude Code.

## Regra central

Cada fase mapeada a um `skill_id` pela configuração embutida do MCP roda em sub-agent foreground, blocking, um por vez.

A primeira ação do sub-agent deve ser carregar o `SKILL.md` completo do `skill_id` resolvido. Depois disso ele executa a skill.

Proibido:

- "aja como `<skill_id>`" sem carregar o `SKILL.md`;
- copiar resumo da skill para substituir a leitura real;
- executar plano/código/review no fio do orquestrador.

Equivalente aceito: mecanismo nativo do host que injete a skill completa no sub-agent.

## Codex

No Codex, ativar `$atlas-task-validator` carrega skill no contexto atual, mas **não** cria isolamento frio. Para validação G4, o orquestrador deve chamar explicitamente o custom agent nativo:

```text
spawn_agent(agent_type: "atlas-task-validator", items: [{ type: "text", text: "<state_path>" }])
```

O registro ativo desse agent vive em `CODEX_HOME/agents/atlas-task-validator.toml` após `npx github:pauloborini/atlas-workflow init codex`; o bundle mantém `.codex/agents/` como fonte gerada. O arquivo é gerado por `build/gen-host-agent.mjs` com `model = "gpt-5.4"` e `model_reasoning_effort = "high"`. Se o host responder `unknown agent_type`, a fase bloqueia (`blocked`/fail-closed). Proibido fallback para `default`, `$atlas-task-validator`, execução inline ou validação no fio do orquestrador.

## Payload mínimo

- `skill_id`
- `skill_md_path` ou indicação nativa equivalente da skill
- `prd_path` / `plan_path` / flags da fase
- instrução: "primeira ação: carregar a skill completa; segunda ação: executar a skill"

## Descoberta de `SKILL.md`

1. Mecanismo nativo de skills do host.
2. Diretório local de skills do host.
3. Workspace, apenas como fallback de descoberta.

Se não encontrar a skill `atlas-*` exigida, abortar. Não trocar por skill nativa ou variante antiga. Não emular inline.
