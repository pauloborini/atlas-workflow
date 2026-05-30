# Atlas Workflow Orchestrator

Orquestra pipelines completos de desenvolvimento de features no projeto Atlas, automatizando a sequência de skills (PRD generation → planejamento → execução → review) sob demanda.

## Quick Start

```bash
/workflow claude full backlog-item "S05"
```

Pipeline completo executado automaticamente:
1. Gera PRD para sprint S05
2. Valida PRD (detecta ambiguidades automaticamente)
3. Executa entrevista se houver decisões em aberto
4. Cria plano
5. Executa plano
6. (Opcional) Executa review

## Sintaxe

```
/workflow <tool> <mode> <input-type> [flags]
```

### Tools

- `claude` — Claude (MVP)
- `cursor` — Cursor (futuro)
- `codex` — Codex (futuro)

### Modes

- `full` — Pipeline completo (PRD → plano → executor → review opcional)
- `direct` — Pipeline enxuto (PRD → executor → review opcional)
- `interview-only` — Entrevista direta (brainstorm, resolução de decisões)

### Input Types

- `backlog-item` — Sprint ID (ex: S05) ou indicação direta
- `idea` — Indicação/brainstorm curto
- `prd` — Path para PRD existente
- `brainstorm` — Texto livre (só para interview-only)

### Flags

- `--interview` — Força entrevista de PRD mesmo sem ambiguidades
- `--review` — Executa slice-review ao final
- `--help` — Mostra sintaxe completa

## Exemplos

### Full pipeline com sprint S05

```
/workflow claude full backlog-item "S05"
```

Output:
```
✅ Workflow: claude full backlog-item completed

📄 PRD: /path/to/PRD_S05_login.md
📋 Plan: /path/to/PLAN_S05_login.md
🚀 Output: [summary do executor]

Status:
  ✅ PRD valid
  ✅ Ambiguidades resolvidas (2 decisões coletadas)
  ✅ Plano generated
  ✅ Executor output ready
  ⏭️  Slice review: not executed
```

### Direct pipeline com PRD existente + review

```
/workflow claude direct prd "/path/to/PRD_S05.md" --review
```

### Entrevista de brainstorm

```
/workflow claude interview-only brainstorm "Que tal adicionar dark mode?"
```

### Force entrevista mesmo sem ambiguidades

```
/workflow claude full idea "melhorar performance" --interview
```

## Como funciona

### Full Mode

```
1. Parse input (resolve sprint/indicação)
   ↓
2. Generate PRD (claude-sprint-prd-generator)
   ↓
3. Validate PRD (busca TBD, "a confirmar", gaps)
   ↓
4. Interview (automático se ambiguidades OU --interview)
   └─ Atualiza PRD com decisões coletadas
   ↓
5. Plan (claude-plan-handoff)
   ↓
6. Validate Plan (tem gaps?)
   └─ Pergunta: volta? continua TBD? adia?
   ↓
7. Execute (claude-plan-execute com task-validator sub-agent)
   ↓
8. Review (se --review)
   └─ claude-slice-review
   ↓
9. Output (resumo + próximos passos)
```

### Direct Mode

```
1. Parse/Generate PRD
   ↓
2. Validate PRD + Interview (condicional)
   ↓
3. Execute
   ↓
4. Review (se --review)
   ↓
5. Output
```

### Interview-Only Mode

```
1. Entrevista direta (sem PRD anterior)
   ↓
2. Output (PRD esboço + decisões)
```

## Validação automática

Plugin detecta ambiguidades em:
- **Objetivo (§3):** TBD, "a confirmar", vago
- **Escopo (§4):** incompleto, "depende de"
- **Decisões (§5):** vazio ou muito vago
- **Experiência (§8):** gaps, "a definir"
- **Contratos (§9):** "ainda não definido", "mock"

Se encontra ambiguidades → dispara `claude-prd-interview` automaticamente.

## Lógica de decisão

Quando há decisões pendentes:

```
Plugin: Tenho decisões em aberto:
  Q-XXX-01: [decisão 1]
  Q-XXX-02: [decisão 2]

Opções:
  A) Volta pra resolver tudo (roda interview agora)
  B) Continua com recomendações (marca TBD)
  C) Adia essas decisões
```

Você escolhe A/B/C → pipeline continua conforme.

## Integração com seu workflow

### Antes de rodar workflow

1. Análise de sprints futuras
2. Preenchimento de `PERGUNTAS_EM_ABERTO.md` (fora do plugin)
3. Rodada de `open-questions-interview` skill (se necessário)

### Ao rodar workflow

```
/workflow claude full backlog-item "S05"
```

Plugin automatiza tudo. Você valida output.

### Depois de workflow

1. Validação de output do executor
2. (Opcional) Rodada de slice-review: `/workflow claude slice-review /path/to/output`
3. Avança para S06

## Skills envolvidas (Claude MVP)

| Skill | Função |
|-------|--------|
| `claude-sprint-prd-generator` | Gera PRD a partir de sprint/indicação |
| `claude-prd-interview` | Entrevista de PRD (resolve ambiguidades) |
| `claude-plan-handoff` | Cria plano executável |
| `claude-plan-execute` | Executa plano (com task-validator sub-agent) |
| `claude-slice-review` | Review fria de implementação |

## Configuração

Plugin usa `atlas_workflows_config.md` para:
- Mapeamento tool → skills
- Validadores de ambiguidade
- Sequências por modo

Sem config → usa defaults (Claude skills).

## Error handling

- **Sprint não encontrado:** reporta sprints disponíveis
- **Skill falha:** para, reporta erro, oferece retry/skip/abort
- **PRD inválido:** reporta sections faltando
- **Ambiguidades não resolvidas:** pergunta próximos passos

## Próximas versões

- **v0.2** Cursor support
- **v0.3** Codex support
- **v0.4** Antigravity support
- **v1.0** Full feature parity + smart tool detection

## Dúvidas?

Veja `atlas_workflows_config.md` para detalhes técnicos e mapeamentos completos.

---

**Plugin version:** 0.1.0 (MVP)  
**Author:** Paulo Borini  
**Last updated:** 2026-05-30
