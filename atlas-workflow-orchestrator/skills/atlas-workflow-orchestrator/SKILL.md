---
name: atlas-workflow-orchestrator
description: "Orquestra pipeline completo de desenvolvimento de features: /workflow <tool> <mode> <input-type> [flags]. Automatiza PRD generation → validação → entrevista (se necessário) → planejamento → execução → review (opcional)."
category: Development Automation
---

# Atlas Workflow Orchestrator

Orquestra pipelines de desenvolvimento de features no projeto Atlas, automatizando a sequência de skills sob demanda com um único comando.

## Sintaxe

```
/workflow <tool> <mode> <input-type> [flags]
```

### Ferramentas

- `claude` (MVP)
- `cursor` (futuro)
- `codex` (futuro)
- `antigravity` (futuro)

### Modos

- **`full`** — pipeline completo: PRD → validação → entrevista (se necessário) → plano → executor → review (opcional)
- **`direct`** — pipeline enxuto: PRD → validação → entrevista (se necessário) → executor → review (opcional)
- **`interview-only`** — entrevista direta (ex: brainstorm, resolução de decisões)

### Input Types

- **`backlog-item`** — Sprint ID (ex: S05) ou indicação direta (ex: "implementar login")
- **`idea`** — Indicação/brainstorm curto
- **`prd`** — Path para PRD existente ou nome do arquivo
- **`brainstorm`** — Texto livre (só para `interview-only`)

### Flags

- `--interview` — força entrevista de PRD mesmo sem ambiguidades detectadas
- `--review` — executa slice-review ao final (senão é opcional)
- `--help` — mostra sintaxe completa

## Exemplos

```
/workflow claude full backlog-item "S05"
→ Gera PRD para S05, valida, entrevista se necessário, cria plano, executa

/workflow claude direct prd "/path/to/PRD_S05.md" --review
→ Valida PRD, executa direto, roda review ao final

/workflow claude full idea "melhorar performance de listagem" --interview
→ Gera PRD de indicação, força entrevista, plano, executor

/workflow claude interview-only brainstorm "que tal dark mode?"
→ Entrevista direto, sem PRD prévio
```

## Fluxo de execução

### Full mode

1. **Parse input** — resolve backlog-item/idea para contexto de sprint
2. **Generate PRD** — dispara `claude-sprint-prd-generator` (ou equivalente)
3. **Validate PRD** — busca ambiguidades (TBD, "a confirmar", gaps em seções críticas)
4. **Interview (condicional)** — se ambiguidades OU `--interview`, dispara `claude-prd-interview`
5. **Plan** — dispara `claude-plan-handoff`
6. **Validate plan** — se gaps → pergunta (continua com TBD? volta? adia?)
7. **Execute** — dispara `claude-plan-execute` (com `task-validator` como sub-agent)
8. **Review (condicional)** — se `--review`, dispara `claude-slice-review`
9. **Output** — resumo com próximos passos

### Direct mode

1. Parse / Generate PRD (se necessário)
2. Validate PRD → Interview (condicional)
3. Execute → Review (condicional)

### Interview-only mode

1. Entrevista direta (sem PRD anterior)
2. Gera PRD esboço (opcional)

## Validação automática de PRD

Plugin detecta ambiguidades quando seção contém:
- Seção 3 (Objetivo): TBD, "a confirmar", vago, "talvez"
- Seção 4 (Escopo): incompleto, "depende de"
- Seção 5 (Decisões): vazio ou muito vago
- Seção 8 (Experiência): gaps, "a definir"
- Seção 9 (Dados/contratos): "ainda não definido", "mock"

Se encontra N ambiguidades → dispara `claude-prd-interview` automaticamente (a menos que tenha certeza).

## Lógica de decisão

Quando há decisões pendentes durante entrevista ou validação de plano:

```
Plugin: "Tenho decisões em aberto:"
  Q-XXX-01: [decisão 1]
  Q-XXX-02: [decisão 2]

Opções:
  A) Volta pra resolver tudo (roda interview agora)
  B) Continua com recomendações (marca TBD, segue)
  C) Adia essas decisões
```

Usuário escolhe A/B/C → plugin continua conforme.

## Output

```
✅ Workflow: claude full backlog-item completed

📄 PRD: /path/to/PRD_S05_login.md
📋 Plan: /path/to/PLAN_S05_login.md
🚀 Output: [summary 1-2 linhas do executor]

Status:
  ✅ PRD valid
  ✅ Ambiguidades resolvidas
  ✅ Plano generated
  ✅ Executor output ready
  ⏭️  Slice review: not executed (run with --review next time or manually)

Próximo passo:
  [ ] Validar executor output
  [ ] Rodar slice-review (opcional)
  [ ] Avançar para S06
```

## Integração com PERGUNTAS_EM_ABERTO.md

Plugin verifica `PERGUNTAS_EM_ABERTO.md` durante validação de PRD. Se houver Q-… abertas relacionadas à sprint → dispara `claude-open-questions-interview` para condensar respostas (fora do pipeline automatizado).

## Error handling

- **Sprint não encontrado** → reporta sprints disponíveis
- **Skill falha** → para, reporta erro, oferece retry/skip/abort
- **PRD inválido** → reporta sections faltando, opção de continuar com warning
- **Ambiguidades não resolvidas** → pergunta próximos passos (ver Lógica de decisão)

## Skills envolvidas (Claude MVP)

| Skill | Entrada | Saída |
|-------|---------|-------|
| `claude-sprint-prd-generator` | sprint_id/indicação | prd_path, decisions_found |
| `claude-prd-interview` | prd_path, ambiguities | prd_updated_path, decisions |
| `claude-plan-handoff` | prd_path | plan_path |
| `claude-plan-execute` | plan_path | executor_output, evidence |
| `claude-slice-review` | executor_output | review_feedback |

**Sub-agent:** `task-validator` (dentro de execute)

## Configuração

Plugin referencia `atlas_workflows_config.md` para:
- Mapeamento tool → skills
- Validadores de ambiguidade
- Sequências de skill por modo

Se não houver config → usa defaults (Claude skills).

## Próximas fases

- **v0.2** Cursor support
- **v0.3** Codex support
- **v0.4** Antigravity support
- **v1.0** Full feature parity em todas as ferramentas
