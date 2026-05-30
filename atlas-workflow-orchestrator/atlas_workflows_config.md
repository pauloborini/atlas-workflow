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
    - prd_generator          # artefato: PRD_*.md          (gate G1)
    - validate_prd           # scan determinístico         (gate G5)
    - prd_interview (if ambiguidades OR --interview flag)   # artefato: PRD_*.md atualizado (gate G1)
    - plan_handoff           # artefato: PLAN_*.md          (gate G1+G2)
    - validate_plan
    - plan_execute (com task-validator frio via sub-agent)  # artefato: diff + relatório validador (gate G3+G4)
    - slice_review (if --review flag)
  required_artifacts: [PRD_*.md, PLAN_*.md, code_diff, validator_report]
  hard_gates: [G1, G2, G3, G4, G5, G6, G7, G8]
  subagent_order: [prd_generator, plan_handoff, plan_execute → task-validator, slice_review (if --review)]

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
    - prd_generator (ou usa existente)   # artefato: PRD_*.md (gate G1)
    - validate_prd                       # scan determinístico (gate G5)
    - prd_interview (if ambiguidades OR --interview flag)
    - plan_execute (sem handoff, task-validator frio via sub-agent)  # artefato: diff + relatório (gate G3+G4)
    - slice_review (if --review flag)
  required_artifacts: [PRD_*.md, code_diff, validator_report]
  hard_gates: [G1, G3, G4, G5, G6, G7, G8]   # G2 não se aplica: direct não produz PLAN_*.md por design
  subagent_order: [prd_generator, plan_execute → task-validator, slice_review (if --review)]
  nota: "se o escopo exigir handoff formal, avisar usuário e sugerir full — nunca fabricar PLAN_*.md ad hoc"
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

  # Gate G5 — decisão determinística, sem escape hatch.
  # Pular entrevista SÓ é válido se scan retornar 0 padrões E o resultado for logado no output.
  # Não existe "pular porque tenho certeza". --interview sempre força.
  skip_rule: "skip_only_if(matches == 0) AND log('Ambiguity scan: 0 padrões — entrevista pulada')"
  force_flag: "--interview"
```

---

## Gates Duros (HARD GATES)

Regras inegociáveis aplicadas pela SKILL. Violação = parar, não contornar.

```yaml
hard_gates:
  G1_artifact_before_advance:
    rule: "fase só conclui se o arquivo que ela produz existir em disco; verificar com Read/ls, nunca auto-relato"
    applies: all
  G2_no_code_before_plan:
    rule: "em full, proibido escrever código (Dart) antes de PLAN_*.md validado existir; sem plano = usar direct"
    applies: full
  G3_real_skill_invocation:
    rule: "cada fase invoca a skill via Skill tool (validador via Agent tool); proibido emular/absorver inline (plano no §10 do PRD NÃO substitui PLAN_*.md)"
    applies: all
  G4_cold_validator:
    rule: "task-validator roda em contexto isolado (sub-agent), recebe git diff + plano; executor não valida o próprio trabalho"
    applies: execution
  G5_deterministic_scan:
    rule: "ver validation.skip_rule — pular entrevista só com 0 padrões logados"
    applies: prd_validation
  G6_verified_status:
    rule: "✅ só após confirmar artefato em disco; artefato exigido ausente => status 'incomplete', nunca 'completed'"
    applies: output
  G7_forced_subagent_dispatch:
    rule: "plan_handoff e plan_execute despachados como sub-agent (Agent tool), NUNCA no fio do orquestrador; PLAN_*.md deve conformar ao template plan_handoff (§2 invariantes, §10 contratos, §11 riscos, §14 checklist, tasks T01..Tn)"
    applies: [plan, execution]
    rationale: "GF07 não disparou sub-agent p/ plano => plano sem template, terrível"
  G8_validation_order:
    rule: "task-validator ANTES (dentro/antes do relatório do executor); slice-review POR ÚLTIMO (só após executor 100%); JAMAIS em paralelo"
    applies: [validation, review]
    rationale: "GF07 rodou task-validator e slice-review concorrentes — funções e ordem distintas"
```

---

## Pré-flight (Fase 0)

```yaml
preflight:
  steps:
    - parse_args                # inválido/--help => mostra sintaxe e para
    - resolve_skills_for_host   # claude-* | cursor-* | codex-* conforme host real
    - verify_subagent_dispatch  # skills despacháveis via Agent tool neste host?
    - declare_execution_plan    # modo, fases, ORDEM dos sub-agents, artefatos, gates
  host_resolution:
    - "host claude-* invocável  => usa claude-*"
    - "host Cursor/Codex        => mapeia cursor-*/codex-* e despacha ESSAS como sub-agent"
  on_missing_skill:
    action: "ABORTAR e reportar"
    forbidden_fallbacks:
      - "implementação direta inline"
      - "contratos equivalentes no fio do orquestrador"
    rationale: "GF07 contornou via 'implementação direta com gates' — fallback inline proibido (G7)"
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
