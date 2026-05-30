# Atlas Workflows Configuration

Mapeamento de ferramentas, modos, skills e validadores para o plugin Atlas Workflow Orchestrator.

---

## Ferramentas e Skills

### Claude (MVP)

```yaml
claude:
  prd_generator: claude-sprint-prd-generator
  prd_interview: claude-prd-interview
  plan_handoff: claude-plan-handoff
  plan_execute: claude-plan-execute
  slice_review: claude-slice-review
  task_validator: (sub-agent dentro de execute)
```

### Cursor (Futuro)

```yaml
cursor:
  prd_generator: (usar codex-sprint-prd-generator ou claude-sprint-prd-generator)
  prd_interview: cursor-prd-interview
  plan_handoff: cursor-plan-handoff
  plan_execute: cursor-plan-execute
  slice_review: cursor-slice-review
  task_validator: (sub-agent dentro de execute)
```

### Codex (Futuro)

```yaml
codex:
  prd_generator: sprint-prd-generator (ou codex-sprint-prd-generator)
  prd_interview: (não tem — usar claude-prd-interview)
  plan_handoff: codex-plan-handoff
  plan_execute: codex-plan-execute
  slice_review: codex-slice-review
  task_validator: (sub-agent dentro de execute)
```

---

## Modos e Sequências

### Full Mode

Sequência completa: PRD generation → validação → entrevista (condicional) → plano → executor → review (condicional)

```yaml
full:
  sequence:
    - prd_generator
    - validate_prd
    - prd_interview (if ambiguidades OR --interview flag)
    - plan_handoff
    - validate_plan
    - plan_execute (com task-validator)
    - slice_review (if --review flag)
  
  decision_on_plan_gap:
    - option_a: volta_para_entrevista
    - option_b: continua_com_recomendacoes (TBD marcado)
    - option_c: adia_decisoes
```

### Direct Mode

Sequência enxuta: PRD → validação → entrevista (condicional) → executor → review (condicional)

```yaml
direct:
  sequence:
    - prd_generator (ou usa existente)
    - validate_prd
    - prd_interview (if ambiguidades OR --interview flag)
    - plan_execute (sem handoff)
    - slice_review (if --review flag)
```

### Interview-Only Mode

Entrevista direta:

```yaml
interview_only:
  sequence:
    - prd_interview (direto, sem PRD anterior)
  
  output:
    - prd_draft (se aplicável)
    - decisions_resolved
```

---

## Validadores de Ambiguidade (PRD)

Plugin escaneia PRD para detectar ambiguidades automaticamente:

```yaml
validation:
  prd_ambiguity_patterns:
    section_3_objective:
      - "TBD"
      - "a confirmar"
      - "talvez"
      - "não definido"
    
    section_4_scope:
      - "pode ser"
      - "depende de"
      - "ainda não"
      - "incompleto"
    
    section_5_decisions:
      - "(empty or minimal content)"
      - "vago"
    
    section_8_experience:
      - "a definir"
      - "gap"
      - "depende de"
    
    section_9_contracts:
      - "ainda não definido"
      - "mock apenas"
      - "a confirmar"
  
  # Se encontra >= 1 padrão, dispara entrevista automaticamente
  threshold: 1
```

---

## Input Types e Resolução

### backlog-item

```yaml
backlog_item:
  input_format: 
    - sprint_id (ex: "S05", "S12")
    - indicacao_direta (ex: "implementar login com 2FA")
  
  resolution:
    - Se sprint_id: busca no BACKLOG_MESTRE.md
    - Se indicação: cria contexto de sprint implícito
  
  output:
    - sprint_context { sprint_id, title, phase, dependencies }
```

### idea

```yaml
idea:
  input_format: "indicação curta ou brainstorm"
  
  output:
    - idea_context { description, scope_estimate }
```

### prd

```yaml
prd:
  input_format:
    - "/path/to/PRD_SXX_slug.md"
    - "PRD_SXX_slug.md" (busca relativo)
  
  resolution:
    - Valida se arquivo existe
    - Valida se tem estrutura PRD_TEMPLATE.md
  
  output:
    - prd_path (absoluto ou relativo resolvido)
```

### brainstorm

```yaml
brainstorm:
  input_format: "texto livre"
  
  allowed_modes: ["interview-only"]
  
  output:
    - brainstorm_text (passado direto pra prd-interview)
```

---

## Flags

```yaml
flags:
  interview:
    description: "Força prd-interview mesmo sem ambiguidades"
    type: "boolean"
    default: false
  
  review:
    description: "Executa slice-review ao final"
    type: "boolean"
    default: false
  
  help:
    description: "Mostra sintaxe e exemplos"
    type: "boolean"
    default: false
```

---

## Output Standard

Todos os modos retornam:

```
✅ Workflow: <tool> <mode> <input-type> completed

📄 PRD: <path>
📋 Plan: <path> (se aplicável)
🚀 Output: <summary 1-2 linhas>

Status:
  ✅/❌ PRD valid
  ✅/❌ Ambiguidades (resolvidas/pendentes)
  ✅/❌ Plano generated (se aplicável)
  ✅/❌ Executor output ready
  ⏭️  Slice review: (executed/not executed)

Próximo passo:
  - [ ] Validar output
  - [ ] Rodar slice-review (se necessário)
  - [ ] Avançar para próxima sprint
```

---

## Integração com PERGUNTAS_EM_ABERTO.md

Durante validação de PRD, plugin verifica:

```
PERGUNTAS_EM_ABERTO.md
  ↓
  Tem Q-… abertas para esta sprint?
  ├─ SIM: informa ao usuário, sugere rodar open-questions-interview
  │       (fora do pipeline automatizado)
  └─ NÃO: continua
```

---

## Error Handling Standard

| Erro | Ação |
|------|------|
| Sprint não encontrado | Reporta sprints disponíveis, pede confirmação |
| Arquivo PRD não existe | Reporta path tentado, oferece criar novo |
| Skill falha | Para, mostra erro, oferece retry/skip/abort |
| PRD inválido (mal formatado) | Reporta sections faltando, opção continuar com warning |
| Ambiguidades não resolvidas | Pergunta: volta? continua TBD? adia? |
| Network/timeout | Retry automático com backoff, depois reporta |

---

## Próximas fases de expansão

- **v0.2** Cursor support (mesmo config, skills do cursor)
- **v0.3** Codex support (sem prd-interview nativa)
- **v0.4** Antigravity support
- **v1.0** Full parity, smart tool detection
