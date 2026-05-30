---
name: atlas-workflow-orchestrator
description: "Orquestra pipeline completo de desenvolvimento de features: /workflow <tool> <mode> <input-type> [flags]. Automatiza PRD generation → validação → entrevista (se necessário) → planejamento → execução → review (opcional). Pipeline orientado a artefato com gates duros: cada fase só conta se produzir arquivo verificável em disco."
category: Development Automation
---

# Atlas Workflow Orchestrator

Orquestra pipelines de desenvolvimento de features no projeto Atlas, automatizando a sequência de skills sob demanda com um único comando.

> **v0.1.2 — pipeline orientado a artefato, não a intenção.** Cada fase do pipeline só é considerada concluída se produzir um **arquivo verificável em disco**. A próxima fase lê esse arquivo. Sem artefato → a fase não aconteceu → o pipeline **bloqueia**. As skills do pipeline são invocadas de verdade (via Skill tool / sub-agent), **nunca emuladas inline**. Esta é a regra que impede `full` de degradar para `direct` ou para "só coda".

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

- **`full`** — pipeline completo: PRD → validação → entrevista (se necessário) → **plano (artefato obrigatório)** → executor → review (opcional)
- **`direct`** — pipeline enxuto: PRD → validação → entrevista (se necessário) → executor → review (opcional). **Não produz plano de handoff** — a diferença real para `full` é exatamente essa.
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
→ Gera PRD para S05, valida, entrevista se necessário, cria PLAN_*.md, executa a partir do plano

/workflow claude direct prd "/path/to/PRD_S05.md" --review
→ Valida PRD, executa direto (sem handoff), roda review ao final

/workflow claude full idea "melhorar performance de listagem" --interview
→ Gera PRD de indicação, força entrevista, plano, executor

/workflow claude interview-only brainstorm "que tal dark mode?"
→ Entrevista direto, sem PRD prévio
```

---

## Fase 0 — Pré-flight obrigatório (antes de qualquer fase)

Executar **antes** de iniciar o pipeline. Se qualquer item falhar, **parar e reportar** — nunca emular.

1. **Parse** dos argumentos `<tool> <mode> <input-type> [input] [flags]`. Se inválido ou `--help` → mostrar sintaxe e parar.
2. **Resolver as skills** do `<tool>` via `atlas_workflows_config.md` (ex: `claude` → `claude-plan-handoff`, `claude-plan-execute`, etc.).
3. **Verificar invocabilidade.** Confirmar que as skills mapeadas existem como skills invocáveis neste host (Skill tool disponível e a skill listada). Se uma skill exigida pelo modo **não** existir como invocável:
   ```text
   ⛔ Pré-flight falhou
      Host: <host detectado>
      Skill exigida ausente: <nome>
      Motivo: skill não está disponível como invocável neste host
      Ação: rodar em host que tenha as skills do <tool>, ou trocar de <tool>
   ```
   **Não** emular a skill inline. Emulação inline é a falha-raiz que esta versão proíbe.
4. **Declarar o plano de execução** (1 bloco curto): modo, sequência de fases, artefatos esperados por fase, gates duros aplicáveis. Só então iniciar a Fase 1.

---

## Gates duros (HARD GATES)

Regras inegociáveis. Violação = parar, não contornar.

| # | Gate | Aplica a |
|---|------|----------|
| G1 | **Artefato antes de avançar.** Uma fase só conta como concluída se o arquivo que ela produz existir em disco. Verificar com leitura real do arquivo (Read/ls), nunca por auto-relato. | todas |
| G2 | **Em `full`, proibido escrever qualquer código (Dart) antes de existir `PLAN_*.md` validado em disco.** Se for escrever código sem plano, o modo correto é `direct` — então pare e avise o usuário do mismatch. | `full` |
| G3 | **Skills invocadas de verdade.** Cada fase invoca a skill correspondente via Skill tool (ou sub-agent via Agent tool para o validador). Proibido absorver o trabalho da skill no mesmo turno "implicitamente" (ex: plano dentro do §10 do PRD não substitui `PLAN_*.md`). | todas |
| G4 | **Validador frio é sub-agent separado.** O `task-validator` roda em contexto isolado (Agent tool), recebendo o git diff da slice + o plano. O executor **não** valida o próprio trabalho no mesmo contexto. | execução |
| G5 | **Scan de ambiguidade determinístico e logado.** A decisão de pular a entrevista só é válida se o scan retornar **zero** padrões e esse resultado for registrado no output. Não existe "pular porque tenho certeza". `--interview` sempre força. | validação PRD |
| G6 | **Status verificado, não auto-reportado.** O ✅ de cada item no output só pode ser marcado após confirmar o artefato em disco. Faltou artefato exigido pelo modo → status final `incomplete`, nunca `completed`. | output |

---

## Fluxo de execução

### Full mode

Artefatos esperados (em ordem): `PRD_*.md` → (`PRD_*.md` atualizado) → `PLAN_*.md` → diff de código → relatório do validador.

1. **Parse input** — resolve backlog-item/idea para contexto de sprint.
2. **Generate PRD** — invoca `claude-sprint-prd-generator`. **Gate G1:** confirmar `PRD_*.md` em disco antes de seguir.
3. **Validate PRD** — roda o scan de ambiguidade (ver "Validação automática"). **Gate G5:** registrar quantos padrões foram encontrados.
4. **Interview (condicional)** — se ambiguidades ≥ 1 **OU** `--interview` → invoca `claude-prd-interview`. Atualiza o `PRD_*.md` com as decisões. **Gate G1:** confirmar PRD atualizado.
5. **Plan** — invoca `claude-plan-handoff`. **Gate G1 + G2:** confirmar `PLAN_*.md` em disco. **Nenhuma linha de código pode ter sido escrita até aqui.**
6. **Validate plan** — se há gaps → aplica a Lógica de decisão (A/B/C).
7. **Execute** — invoca `claude-plan-execute`, lendo o `PLAN_*.md`. **Gate G4:** o `task-validator` roda como sub-agent frio (Agent tool) com o git diff + plano antes do relatório final.
8. **Review (condicional)** — se `--review` → invoca `claude-slice-review`.
9. **Output** — ledger verificado (ver "Output") + próximos passos.

### Direct mode

Artefatos esperados: `PRD_*.md` → (atualizado) → diff de código → relatório do validador. **Sem `PLAN_*.md`** — por design.

1. Parse / Generate PRD (se necessário). **Gate G1.**
2. Validate PRD → Interview (condicional). **Gate G5.**
3. Execute — invoca `claude-plan-execute` direto a partir do PRD. **Gate G4** (validador frio).
4. Review (condicional).
5. Output (ledger verificado).

> Se durante `direct` o escopo exigir um plano de handoff formal, **avise o usuário** e sugira `full` — não fabrique um `PLAN_*.md` ad hoc no meio de `direct`.

### Interview-only mode

1. Entrevista direta (sem PRD anterior) — invoca `claude-prd-interview`.
2. Gera PRD esboço (opcional).

---

## Validação automática de PRD

O scan é **determinístico**. Marca ambiguidade quando uma seção contém qualquer padrão abaixo (lista canônica em `atlas_workflows_config.md`):

- **§3 Objetivo:** `TBD`, `a confirmar`, `talvez`, `não definido`
- **§4 Escopo:** `pode ser`, `depende de`, `ainda não`, `incompleto`
- **§5 Decisões:** vazio/conteúdo mínimo, `vago`
- **§8 Experiência:** `a definir`, `gap`, `depende de`
- **§9 Dados/contratos:** `ainda não definido`, `mock apenas`, `a confirmar`

**Threshold = 1.** Se ≥ 1 padrão → dispara `claude-prd-interview`. **Gate G5:** se 0 padrões, registrar `Ambiguity scan: 0 padrões — entrevista pulada` no output. Não há decisão subjetiva de "tenho certeza, pulo".

---

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

---

## Output

O ledger é **verificado contra disco** (Gate G6). Cada artefato listado precisa existir.

```
✅ Workflow: claude full backlog-item completed

📄 PRD: /path/to/PRD_S05_login.md            [verificado em disco]
📋 Plan: /path/to/PLAN_S05_login.md          [verificado em disco]
🚀 Output: [summary 1-2 linhas do executor]

Status:
  ✅ PRD valid
  ✅ Ambiguity scan: 2 padrões → entrevista executada (2 decisões)
  ✅ Plano generated (PLAN_*.md presente)
  ✅ Executor output ready
  ✅ Validador frio: P1=0 P2=1 P3=2 (sub-agent task-validator)
  ⏭️  Slice review: not executed (run with --review)

Próximo passo:
  [ ] Validar executor output
  [ ] Rodar slice-review (opcional)
  [ ] Avançar para próxima sprint
```

Se algum artefato exigido pelo modo estiver ausente, o cabeçalho vira:

```
⚠️  Workflow: claude full backlog-item incomplete
   Faltando: PLAN_*.md (Gate G2 bloqueou execução de código)
```

---

## Integração com PERGUNTAS_EM_ABERTO.md

Plugin verifica `PERGUNTAS_EM_ABERTO.md` durante validação de PRD. Se houver Q-… abertas relacionadas à sprint → dispara `claude-open-questions-interview` para condensar respostas (fora do pipeline automatizado).

---

## Error handling

- **Pré-flight falha (skill ausente no host)** → para, reporta, não emula (ver Fase 0).
- **Sprint não encontrado** → reporta sprints disponíveis.
- **Skill falha** → para, reporta erro, oferece retry/skip/abort.
- **PRD inválido** → reporta sections faltando, opção de continuar com warning.
- **Gate duro violado** → para, reporta qual gate (G1–G6) e o artefato/condição faltante.
- **Ambiguidades não resolvidas** → pergunta próximos passos (ver Lógica de decisão).

---

## Skills envolvidas (Claude MVP)

| Skill | Entrada | Saída (artefato) |
|-------|---------|------------------|
| `claude-sprint-prd-generator` | sprint_id/indicação | `PRD_*.md`, decisions_found |
| `claude-prd-interview` | prd_path, ambiguities | `PRD_*.md` atualizado, decisions |
| `claude-plan-handoff` | prd_path | `PLAN_*.md` |
| `claude-plan-execute` | plan_path (full) ou prd_path (direct) | diff de código, evidência |
| `claude-slice-review` | diff/output | review_feedback |

**Sub-agent frio (Gate G4):** `claude-task-validator` (Agent tool, contexto isolado), dentro de execute.

---

## Configuração

Plugin referencia `atlas_workflows_config.md` para:
- Mapeamento tool → skills
- Padrões de ambiguidade (lista canônica)
- Sequências de skill por modo + artefatos esperados
- Gates duros

Se não houver config → usa defaults (Claude skills) e os gates desta SKILL.

---

## Changelog

- **v0.1.2** — Pipeline orientado a artefato. Adicionados: Fase 0 pré-flight (verifica invocabilidade, proíbe emulação inline), Gates duros G1–G6, scan de ambiguidade determinístico (mata o escape hatch "tenho certeza"), validador frio obrigatório como sub-agent, ledger verificado contra disco. `direct` explicitamente não produz `PLAN_*.md`.
- **v0.1.1** — `/workflow` slash command.
- **v0.1.0** — MVP (Claude skills).

## Próximas fases

- **v0.2** Cursor support
- **v0.3** Codex support
- **v0.4** Antigravity support
- **v1.0** Full feature parity em todas as ferramentas
