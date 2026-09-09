# Os 7 Agentes da Esteira

Cada agente é responsável por um estágio do SDLC. Todos rodam no **GitHub Copilot Enterprise** e seguem o mesmo padrão: recebem um artefato aprovado como input, produzem um artefato como output, e têm uma suite de evals que valida seu comportamento em CI.

---

## Agente 01 — Intent

**Estágio:** Plan  
**O que muda:** Ideias saem do BusinessMap como artefato estruturado em minutos, não dias de alinhamento em reuniões. O `intent.md` força a articulação do problema antes da solução.

### Input / Output

| | |
|---|---|
| **Input** | URL ou ID do card no BusinessMap |
| **Output** | `intent.md` — problema declarado, outcome esperado, usuários afetados, sistemas envolvidos, constraints, perguntas abertas, autor e timestamp |
| **Ferramenta** | Copilot Chat + BusinessMap MCP (VS Code agent mode) |

### Fluxo

1. Product Owner abre Copilot Chat com BusinessMap MCP ativo
2. Prompt: `"Converta o card #[ID] do BusinessMap em um intent.md seguindo nosso template"`
3. Copilot chama o MCP, busca dados do card, gera `intent.md`
4. PO revisa e corrige — especialmente as perguntas abertas e constraints
5. Commit do `intent.md` → abre PR
6. Merge = aceito; PR fechado sem merge = rejeitado

O link do card BusinessMap é mantido no frontmatter do `intent.md` — rastreabilidade bidirecional.

### Evals

- **20–50 cards reais** com `intent.md` esperado como ground truth
- Graders (code-based): todos os campos obrigatórios presentes? Link ao card válido? Timestamp e autor preenchidos?
- Graders (model-based): o problema declarado corresponde ao problema do card? Os critérios de sucesso são mensuráveis? As perguntas abertas são relevantes e não triviais?
- Métrica: **pass@1** — geração única deve passar

### Gate de Governança
Product Owner aprova via merge do PR. Sem merge, a ideia não avança.

### Medição
- Leading: Tempo do card → `intent.md` commitado (target: <4h)
- Lagging: Taxa de `intent.md` aceitos sem revisões maiores após o stage de spec

---

## Agente 02 — Spec

**Estágio:** Design  
**Pré-requisito:** Fase 0b ativa — skills `bbds-ux-guidelines` e `bbds-patterns` disponíveis.  
**O que muda:** Requisitos e design colapsam em uma sessão. Políticas aplicadas durante a geração, não descobertas em revisão semanas depois. Componentes BBDS mapeados por tela antes de qualquer linha de código.

### Input / Output

| | |
|---|---|
| **Input** | `intent.md` aprovado + protótipo Figma + skills de política + MCP de catálogo de serviços *(quando disponível)* |
| **Output** | `spec.md` + `api-contract.md` — spec com requisitos, flags de compliance e componentes BBDS; contrato de API com endpoints classificados (reutilizar / extensão / novo) |
| **Ferramenta** | Copilot agent mode + skills de política + skills BBDS |

### Fluxo

1. PO abre Copilot agent mode com `#file:intent.md`
2. Copilot carrega automaticamente: `security`, `platform-standards`, `bbds-ux-guidelines`, `bbds-patterns`
3. *[Quando MCP de catálogo disponível]* Consulta quais endpoints de backend já existem
4. Gera `spec.md` com seção `## Flags` para conflitos/riscos
5. Gera `api-contract.md` com cada endpoint classificado: reutilizar / extensão / novo
6. PO endereça flags com policy owners antes de commitar
7. Commit de `spec.md` + `api-contract.md` → workflow `intent-to-spec.yml` valida schema e flags resolvidas
8. Merge = aprovado

**Nota sobre design visual:** UX + engenheiros continuam criando protótipos no Figma. O `spec.md` documenta os requisitos e mapeia componentes BBDS — não gera mockups. O link do protótipo Figma entra no frontmatter do spec.

### Evals

- **20–50 pares** (`intent.md`, `spec.md` esperado) de projetos reais
- Graders (code-based): critérios de aceitação verificáveis? Seção `## Flags` presente e categorizada? Referência ao `intent.md`? Componentes BBDS listados existem no `bbds-api-reference.md`?
- Graders (model-based): spec endereça todos os pontos do intent? Políticas de segurança aplicadas (não apenas mencionadas)? Componentes BBDS respeitam `use_when`/`avoid_when` do `bbds-ux-guidelines`?
- Métricas: **pass@1**, taxa de flags resolvidas antes do merge

### Gate de Governança
PO aprova; policy owners confirmam que flags foram resolvidas. Versão das skills registrada no frontmatter do `spec.md`.

### Medição
- Leading: Tempo entre `intent.md` e `spec.md` commitados (target: <1 dia)
- Lagging: Rework no spec após o build começar (commits de `spec.md` após `plan.md` existir)

---

## Agente 03 — Plan

**Estágio:** Planning  
**O que muda:** Nenhuma implementação sem plano escrito aprovado. O plan.md faz o engenheiro pensar antes de codar — e documenta as decisões para o reviewer.

### Input / Output

| | |
|---|---|
| **Input** | `spec.md` + `api-contract.md` aprovados + `.github/copilot-instructions.md` do repo-alvo |
| **Output** | `plan.md` — arquivos que mudam, ordem de trabalho, riscos identificados, estratégia de testes, critérios de "pronto"; estrutura do adapter pattern planejada para endpoints sem impl real |
| **Ferramenta** | Copilot agent mode + Plan Mode nativo |

**Pré-requisito por repo:** `.github/copilot-instructions.md` deve existir (gerado na Fase 0 de bootstrap). Um status check no CI bloqueia merge se o arquivo não existir no repo.

### Fluxo

1. Engenheiro abre VS Code no repo do bundle correto
2. Copilot agent mode com `#file:spec.md` + `#file:api-contract.md` + `copilot-instructions.md` carregado automaticamente
3. Prompt: `"Gere um plan.md para implementar esta spec no contexto deste repo"`
4. Copilot analisa o repo, lista arquivos afetados, propõe ordem de trabalho
5. Engenheiro interroga o plano — riscos, alternativas, impacto em outros bundles
6. Itera até o plano estar implementável; commita `plan.md`
7. Implementação só começa após commit do `plan.md`

Git hooks locais (husky/lefthook) podem dar feedback imediato antes do push.

### Evals

- Tasks: specs reais com `plan.md` esperado como ground truth
- Graders (code-based): arquivos listados no plan existem no repo? Seção de testes presente? Riscos documentados? Nenhum arquivo de segurança listado sem justificativa?
- Graders (model-based): plano executável por engenheiro novo no bundle? Ordem de trabalho faz sentido técnico? Escopo dentro do spec (nem mais, nem menos)?
- Métrica: **pass@1**

### Gate de Governança
Engenheiro commita o `plan.md` explicitamente — esse é o gate. Sem `plan.md`, CI bloqueia merge via status check.

### Medição
- Leading: Proporção de PRs com `plan.md` commitado (target: 100% após rollout)
- Lagging: Divergência entre `plan.md` e diff final do PR

---

## Agente 04 — Build

**Estágio:** Implementação  
**Pré-requisito:** Fase 0b ativa — skill `bbds-api` disponível e `figma-to-code` skill aprimorada com acesso a `bbds-api-reference.md`.  
**O que muda:** Implementação guiada pelo `plan.md` aprovado. API do BBDS sempre atualizada. Telas mapeadas via `figma-to-code` com validação automática de props.

### Input / Output

| | |
|---|---|
| **Input** | `plan.md` + `api-contract.md` + `.github/copilot-instructions.md` + protótipo Figma (referenciado no spec) |
| **Output** | Código implementado + testes + adapters de backend (interface + mock + index com feature flag) para endpoints sem impl real |
| **Ferramentas** | Copilot agent mode + Copilot Edits + skills BBDS + figma-to-code skill |

### Fluxo

1. Engenheiro abre Copilot Edits com `#file:plan.md` + `#file:api-contract.md` como contexto
2. Para novas telas: invoca `figma-to-code` skill (aprimorada na Fase 0b)
   - Identifica componentes BBDS no protótipo Figma
   - Valida props contra `bbds-api-reference.md` atual
   - Aplica migration paths para props `@deprecated`
   - Emite warning se componente não existe no BBDS (candidato a novo componente)
3. Para cada endpoint em `api-contract.md` classificado como "Novo" ou "Extensão" sem impl real: gera adapter pattern (`*Service.ts` + `*ServiceMock.ts` + `index.ts` com feature flag). Telas importam apenas a interface.
4. Implementação em ordem definida no `plan.md`; skill `bbds-api` carregada automaticamente
5. `npm test` / `jest` — loop de feedback antes do push
6. Se `plan.md` precisar ser alterado: atualizar o arquivo e justificar no commit

O Build agent não pode editar arquivos de teste durante uma run de fix — enforçado via instrução no `copilot-instructions.md` e verificado por status check no GitHub Actions.

### Evals

- Tasks: `plan.md` + estado do repo → código esperado
- Graders (code-based): testes passam? Lint/format pass? Sem `console.log` ou debug code? Naming conventions seguidas? Imports de libs não-aprovadas? Props BBDS existem no `bbds-api-reference.md`? Props `@deprecated` usadas sem migration path aplicado (deve ser zero)?
- Graders (model-based): implementação fiel ao `plan.md`? Novos componentes seguem `bbds-patterns`?
- Métricas: **pass@1** (CI first-pass), **pass^3** (consistência em múltiplas tentativas)

### Gate de Governança
CI green — testes + lint + evals. Platform standards versionados e aprovados pela equipe de plataforma.

### Medição
- Leading: Taxa de CI first-pass em mudanças geradas com agente (target: >80%)
- Lagging: Ciclos de rework por PR vs. média histórica

---

## Agente 05 — Test

**Estágio:** Teste  
**O que muda:** Configuração dos agentes (skills, copilot-instructions) é testada como código. Regressões detectadas antes de chegar ao engenheiro.

### Input / Output

| | |
|---|---|
| **Input** | Code diff + resultados de CI + histórico de evals |
| **Output** | Relatório de pass rate por agente, sugestões de correção para falhas de CI |
| **Ferramentas** | GitHub Actions + Copilot API (análise de falhas) |

### Fluxo

1. Push dispara GitHub Action com suite de testes do repo
2. Para falhas de CI: Copilot analisa o erro e sugere correção em comentário no PR
3. Eval suite roda em paralelo com tasks de `agents/05-test/evals/tasks/`
4. Se pass rate < threshold: PR bloqueado (branch protection rule)
5. Incidentes de produção → novo eval task criado a partir do bug — "não pode acontecer de novo"

**Workflow `eval-suite.yml`:**
- Trigger: push para main/staging ou mudança em `copilot-instructions.md` / `skills/`
- Roda todas as tasks de eval de todos os 7 agentes
- Report de pass rate por agente
- Gera badge e histórico de tendência para acompanhamento semanal

### Evals (meta-eval)

- O agente identifica falhas reais e não falsos positivos?
- Tasks: logs de CI com falhas reais vs. diagnóstico esperado
- Graders: diagnóstico correto? Sugestão de fix resolve o problema? Falsos positivos (deve ser zero)?

### Gate de Governança
Threshold de pass rate (ex: 85%) definido em `eval-suite.yml`. Mudança no threshold requer PR aprovado pelo tech lead.

### Medição
- Leading: Eval pass rate por agente, trend semanal
- Lagging: Regressões capturadas em CI vs. chegadas em produção

---

## Agente 06 — Deploy

**Estágio:** Deploy  
**O que muda:** Review bidirecional. Copilot revisa PRs incoming e endereça comentários de review. Humano aprova a decisão — não substitui o revisor.

### Input / Output

| | |
|---|---|
| **Input** | PR diff + `REVIEW.md` + `spec.md` + `plan.md` |
| **Output** | Review comments (bugs, segurança, compliance), aprovação/bloqueio de CI |
| **Ferramentas** | Copilot Code Review (Enterprise) + GitHub Actions |

### Estrutura do `REVIEW.md`

```markdown
## Passes de Review
1. Bugs e erros de lógica (Important)
2. Segurança / vulnerabilidades (Important)
3. Compliance com spec.md e plan.md (Important)
4. Padrões de plataforma React Native (Nit se cosmético)

## Exclusões
- Arquivos gerados (*.generated.ts, __mocks__)
- Regras já enforçadas por lint/CI

## Limites
- Máximo 10 nits por PR; Important findings: ilimitado
```

### Fluxo

1. PR aberto → Copilot Code Review roda automaticamente (Enterprise feature)
2. Review usa `REVIEW.md` como rubrica e `spec.md` como spec de referência
3. Engenheiro menciona `@copilot` em comentários para endereçar findings
4. Branch protection: requer 1 humano + CI green para merge
5. Agente não pode aprovar o próprio código — separação de funções obrigatória

**Tiering de autonomia:**
- Dev: deploy livre via workflow
- Staging: agente abre PR, CI aprova, deploy automático
- Prod: agente prepara, release manager autoriza via GitHub environment protection (aprovador nomeado)

### Evals

- Tasks: PRs reais com review esperado como ground truth
- Graders (code-based): todos os bugs reais sinalizados? Falsos positivos < 10%? Findings categorizados corretamente?
- Graders (model-based): finding tem evidência clara (linha + explicação)? Sugestão de fix é correta?
- Métricas: precision e recall de findings reais

### Gate de Governança
1 humano + CI green para merge. Prod requer release manager nomeado via GitHub environment protection.

**Gate adicional — prontidão de backend:** Se endpoint em `api-contract.md` ainda sem `*ServiceImpl.ts` → deploy com feature flag desativada obrigatório. Feature flag ativada manualmente pelo release manager quando backend validado em staging.

### Medição
- Leading: Tempo até primeiro review (target: <15min após PR aberto)
- Lagging: Defeitos capturados pre-merge vs. chegados em prod; métricas DORA

---

## Agente 07 — Maintain

**Estágio:** Manutenção  
**O que muda:** Três camadas de monitoramento autônomo convergem num pipeline de correlação. Achados re-entram no pipeline como `intent.md`. Loop se auto-perpetua.

### Input / Output

| | |
|---|---|
| **Input** | Firebase MCP (crashes) + Journey Monitor (jornadas negociais) + bands.yaml (CI/ops) |
| **Output** | GitHub issue (visibilidade imediata) + PR com `intent.md` (entrada formal no pipeline) |
| **Ferramentas** | GitHub Actions (cron) + Firebase MCP + Journey Monitor API + Copilot API |

### Três Fontes de Monitoramento

| Fonte | O que detecta | Localização | Path de análise |
|---|---|---|---|
| Firebase Crashlytics | Crash técnico | Stacktrace → arquivo:linha | Direto: desminifica → root cause |
| Journey Monitor | Falha negocial | Flow + tela problemática | Correlação: crash? endpoint? commit? deploy? |
| bands.yaml | Degradação de qualidade | Métricas de CI, cycle time | Determinístico: Western Electric rules |

### Fluxo — Crashes (Firebase, solução existente + extensão)

1. Cron busca top crashes + novos crash classes via Firebase MCP
2. Para cada crash relevante: desminificação → análise Copilot → hipótese de root cause
3. GitHub issue aberta (comportamento atual) + PR com `intent.md` (extensão nova)
4. On-call: merge PR = entra no pipeline; close = dismiss

### Fluxo — Jornadas (Journey Monitor + pipeline de correlação)

1. Cron consulta Journey Monitor; detecta flows com queda de taxa de sucesso
2. Para cada flow: obtém `flow_id`, `screen_id`, `endpoint_errors`
3. Correlação paralela (steps do GitHub Actions):
   - Firebase MCP: crash na tela problemática no mesmo período?
   - `git log --since=48h -- *<screen_id>*`: commit recente toca essa tela?
   - `gh release list`: deploy recente do bundle?
4. Copilot recebe o correlation bundle e segue a árvore de diagnóstico:
   - `endpoint_error` → problema de backend
   - `crash_found` → bug técnico (path Firebase)
   - `commit_recent` → regressão provável
   - nenhum → silent failure (hipótese aberta)
5. GitHub issue + PR com `intent.md` estruturado

**Exemplo de `intent.md` gerado por falha de jornada:**
```yaml
source: journey-monitor
flow: contratacao-credito
problematic_screen: ConfirmacaoDadosScreen
success_rate_drop: -34% (baseline 7d)
correlation:
  endpoint_errors: "POST /api/v2/credit/validate → 503 (12% error rate)"
  firebase_crash: nenhum na tela afetada
  recent_commit: "abc123 — feat: atualiza validação de crédito (2d atrás)"
  recent_deploy: "v2.4.1 (3d atrás, bundle contratacao)"
diagnosis_path: endpoint_error
hypothesis: >
  Endpoint /api/v2/credit/validate retornando 503 após deploy v2.4.1.
action: ping time de API + avaliar rollback de v2.4.1
```

### Evals

- **Crashes (Firebase):**
  - Stacktrace desminificado → root cause esperado (ground truth: fix posterior)
  - Novo crash class → `intent.md` com campos obrigatórios e problema descrito?
  - Crash known → agente não abre duplicate?
- **Jornadas — um task por branch da árvore de diagnóstico:**
  - Jornada com `endpoint_error` → `diagnosis_path` correto? `action` correto?
  - Jornada com crash correlacionado → diagnóstico usa o stacktrace?
  - Jornada com commit recente, sem crash, sem endpoint → regressão identificada?
  - Jornada sem correlação → agente não fabrica hipótese, mantém aberta?
- Graders (code-based): `intent.md` com todos os campos? `diagnosis_path` válido? Duplicate detection funciona? SLA <30min?
- Graders (model-based): hipótese plausível dado o correlation bundle? Action proporcional à severity?
- Métricas: time-to-intent, `diagnosis_path` accuracy, resolution rate, repeat incident rate

### Gate de Governança
On-call triagem via PR: merge = aceita; close = dismiss. Dismiss tunea a sensibilidade dos bands para aquele sinal específico, evitando alertas repetidos para problemas conhecidos.

### Medição
- Leading: Tempo de detecção → `intent.md` na fila (target: <30min)
- Lagging: % de achados que viram fixes; incidentes repetidos por classe (-50% em 6m); `diagnosis_path` correto vs. root cause real confirmado no post-mortem
