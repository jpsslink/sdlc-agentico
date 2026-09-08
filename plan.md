# Esteira Agêntica AI-Native SDLC — Plano de Implementação

## Context

O objetivo é implantar uma esteira de desenvolvimento AI-native baseada no playbook da Anthropic, adaptada para:
- **Runtime**: GitHub Copilot Enterprise (não Claude Code)
- **Stack**: React Native, multi-bundle, dezenas de equipes/repos sem arquivos MD de contexto
- **Gestão**: Cards do BusinessMap como fonte das ideias (MCP já existe)
- **Design**: Figma (manual, sem dev mode em escala); design ainda criado por UX/engenheiros
- **CI/CD**: Híbrido (GitHub Actions + possivelmente Bitrise para mobile)
- **Maior dor atual**: Conhecimento de plataforma disperso — MD files em repos, Figma, conhecimento tácito

A esteira transforma o ciclo linear (Plan→Design→Build→Test→Deploy→Maintain) em um loop contínuo onde humanos governam, agentes executam.

---

## Mapeamento: Playbook Anthropic → GitHub Copilot Enterprise

| Conceito Anthropic | Equivalente GitHub Copilot Enterprise | Compatível? |
|---|---|---|
| `CLAUDE.md` | `.github/copilot-instructions.md` | Mesmo propósito, nome/local diferente |
| Skills (`.claude/skills/`) | Skills (`.github/skills/` ou `.claude/skills/`) | **Formato SKILL.md idêntico** — funciona em ambos |
| Plan Mode | Plan Mode nativo do Copilot | **Mesmo comportamento** |
| Subagents | `runSubAgent` (`#tool:agent/runSubagent`) | **Equivalente funcional direto** |
| Hooks | GitHub Actions + git hooks (pre-commit/pre-push) | Diferente — Copilot não tem hook system nativo |
| PR Review | Copilot Code Review (Enterprise) | Equivalente |
| Evals em CI | GitHub Actions workflows | Equivalente |
| Managed Settings | Políticas de organização GitHub Copilot Enterprise | Equivalente |

**Implicação crítica:** Skills escritas no formato SKILL.md funcionam tanto em Claude Code quanto em
GitHub Copilot — o formato é compartilhado. O Copilot suporta explicitamente `.claude/skills/` como
diretório válido. A adaptação real se resume a: (1) `CLAUDE.md` → `copilot-instructions.md`,
e (2) substituição do sistema de hooks nativos do Claude Code por GitHub Actions.

---

## Estrutura do Repositório `sdlc-agentico`

Este repo é o meta-repo da esteira — contém templates, agentes, skills, evals e documentação.

```
sdlc-agentico/
├── .github/
│   ├── copilot-instructions.md          # Contexto principal para o Copilot (equiv. CLAUDE.md)
│   ├── skills/                          # Skills compartilhadas (compatível com Copilot E Claude Code)
│   │   ├── platform-standards/
│   │   │   └── SKILL.md
│   │   ├── security/
│   │   │   └── SKILL.md               # OWASP RN, secure storage, cert pinning
│   │   ├── branding/                   # FASE 2
│   │   │   └── SKILL.md
│   │   ├── compliance/                 # FASE 2 (LGPD, WCAG, data residency)
│   │   │   └── SKILL.md
│   │   ├── ux/                        # FASE 2 (design system, navigation patterns)
│   │   │   └── SKILL.md
│   │   ├── bbds-api/                   # FASE 0b: API de componentes BBDS (auto-gerada)
│   │   │   └── SKILL.md
│   │   ├── bbds-ux-guidelines/         # FASE 0b: Guias de uso UX por componente
│   │   │   └── SKILL.md
│   │   └── bbds-patterns/             # FASE 0b: Padrões de composição de telas
│   │       └── SKILL.md
│   └── workflows/
│       ├── bootstrap-repo.yml           # Stage 0: inicializa repo de equipe
│       ├── intent-to-spec.yml           # Stage 2: automação intent→spec
│       ├── copilot-pr-review.yml        # Stage 5: PR review
│       ├── eval-suite.yml               # Stage 4: evals em CI
│       ├── monitor-breach.yml           # Stage 6: monitoramento
│       └── bbds-api-sync.yml           # Fase 0b: regera bbds-api-reference.md ao bump do BBDS
│
├── templates/
│   ├── intent.md                        # Template do artefato intent
│   ├── spec.md                          # Template do artefato spec
│   ├── plan.md                          # Template do artefato plan
│   ├── REVIEW.md                        # Critérios de revisão de PR
│   ├── bands.yaml                       # Config de monitoramento (Stage 6)
│   └── copilot-instructions.md          # Template base para repos de equipe
│
├── agents/
│   ├── 01-intent/                       # Stage 1: Plan
│   │   ├── README.md
│   │   ├── prompt.md                    # Instruções do agente (persona + contexto)
│   │   └── evals/
│   │       ├── tasks/                   # 20-50 tasks de teste
│   │       └── graders/                 # Lógica de avaliação
│   ├── 02-spec/                         # Stage 2: Design
│   ├── 03-plan/                         # Stage 3a: Planning
│   ├── 04-build/                        # Stage 3b: Build
│   ├── 05-test/                         # Stage 4: Test
│   ├── 06-deploy/                       # Stage 5: Deploy
│   └── 07-maintain/                     # Stage 6: Maintain
│
├── platform-knowledge/
│   ├── README.md                        # Guia do bootstrap de conhecimento
│   ├── extraction-guide.md              # Como extrair conhecimento tácito
│   ├── figma-extraction-guide.md        # Como exportar tokens/components do Figma
│   ├── platform-standards-draft.md     # Rascunho a ser validado pela equipe
│   ├── bbds-api-reference.md           # Fase 0b: API BBDS (auto-gerada via ts-morph — não editar)
│   └── bbds-ux-guidelines.yaml         # Fase 0b: Guias de uso UX (curado pelo time de UX)
│
├── evals/
│   ├── README.md
│   └── framework.md                     # Como escrever e rodar evals
│
├── scripts/
│   └── extract-bbds-types.ts           # Fase 0b: ts-morph → bbds-api-reference.md
│
└── docs/
    ├── onboarding.md                    # Como uma equipe adota a esteira
    ├── governance.md                    # Cadeia de aprovação e audit trail
    ├── measurement.md                   # KPIs por stage
    └── legacy-repos.md                  # Como migrar repos existentes
```

---

## Fase 0 — Bootstrap de Conhecimento de Plataforma (CRÍTICO)

**Por quê primeiro:** Sem o conhecimento de plataforma formalizado, todos os agentes
posteriores operam sem contexto — geram código que viola padrões internos.
Este é o maior gargalo atual e deve ser endereçado antes de qualquer outro stage.

### O que fazer

1. **Mining de repos existentes**: GitHub Action que varre todos os repos da equipe de
   plataforma, coleta MD files, extrai convenções e gera um rascunho de
   `platform-standards-draft.md`.

2. **Extração do Figma**: Guia de exportação de design tokens, nomes de componentes
   e padrões de interação. Como a integração via MCP está limitada (licenças dev mode),
   a exportação é manual neste momento: JSON de tokens + documentação de componentes
   exportados para `platform-knowledge/figma-extraction-guide.md`.

3. **Knowledge sessions**: 2-3 sessões estruturadas com a equipe de plataforma para
   capturar conhecimento tácito. Perguntas-padrão em `extraction-guide.md`. O output
   alimenta o Copilot para gerar documentação.

4. **Validação e versionamento**: A equipe de plataforma revisa e aprova
   `platform-standards-draft.md` via PR. Esse arquivo se torna a fonte de verdade para
   a skill em `.github/skills/platform-standards/SKILL.md`.

5. **Bootstrap por repo**: Script/workflow que copia o template de
   `copilot-instructions.md`, injeta o contexto de plataforma, e usa Copilot (plan mode
   + agente) para analisar o repo específico e preencher seções bundle-específicas.
   Resultado vira PR para o time revisar.

### Artefatos resultantes
- `platform-knowledge/platform-standards-draft.md` (aprovado pela equipe de plataforma)
- `.github/skills/platform-standards/SKILL.md` (skill operacional, compatível com Copilot e Claude Code)
- `templates/copilot-instructions.md` (template base para todos os repos de equipe)

---

## Fase 0b — Arquitetura de Conhecimento BBDS (INFRAESTRUTURA)

**Por quê como infraestrutura:** O BBDS (design system React Native) é obrigatório para novos fluxos.
Sem conhecimento estruturado do BBDS, o Agente 02 (Spec) especifica componentes errados e
o Agente 04 (Build) usa props obsoletas ou improvisa fora do design system. Este é um pré-requisito
transversal — deve estar ativo antes dos agentes de Spec e Build entrarem em produção.

**Relação com Fase 0:** Enquanto a Fase 0 formaliza padrões de plataforma genéricos,
a Fase 0b estrutura especificamente o conhecimento do design system — uma camada separada
por ser dinâmica (versiona com cada release do BBDS) e ter duas fontes distintas
(TypeScript types auto-geráveis + guias de UX curados manualmente).

### Três artefatos de conhecimento

#### 1. `platform-knowledge/bbds-api-reference.md` — API auto-gerada
- **Fonte**: TypeScript types + JSDoc do pacote BBDS (lido via ts-morph)
- **Conteúdo**: Para cada componente: props com tipos, defaults, variantes, props `@deprecated` com migration path
- **Manutenção**: Totalmente automática — nunca editar manualmente
- **Gatilho de atualização**: `bbds-api-sync.yml` roda quando `package.json` do BBDS tem version bump

#### 2. `platform-knowledge/bbds-ux-guidelines.yaml` — Guias curados pelo UX
- **Fonte**: Time de UX (curado manualmente, estável entre versões menores)
- **Conteúdo**: Por componente — quando usar, quando não usar, padrões de composição aprovados, anti-patterns
- **Estrutura exemplo**:
  ```yaml
  Button:
    use_when:
      - Ação primária em tela (máximo 1 por tela)
      - Confirmação de formulário
    avoid_when:
      - Navegação — usar Link ou Tab
      - Mais de 3 botões em sequência — usar ActionList
    composition_patterns:
      - Button + Icon: ícone à esquerda do label
    anti_patterns:
      - Botão com label > 4 palavras
  ```
- **Manutenção**: PR com aprovação do UX lead; versionado junto ao repo

#### 3. Três skills resultantes (`.github/skills/`)

| Skill | Conteúdo | Carregada quando |
|---|---|---|
| `bbds-api` | `bbds-api-reference.md` injetado | Qualquer menção a componente BBDS |
| `bbds-ux-guidelines` | `bbds-ux-guidelines.yaml` formatado | Decisões de UX / qual componente usar |
| `bbds-patterns` | Padrões de composição de telas (curado pela plataforma) | Geração de nova tela ou fluxo |

### Pipeline de auto-geração (`scripts/extract-bbds-types.ts`)

```
bbds package (TypeScript) → ts-morph lê types + JSDoc
    → extrai: nome, props, tipos, defaults, @deprecated + migration path
    → merge com bbds-ux-guidelines.yaml (adiciona contexto de uso)
    → gera bbds-api-reference.md em platform-knowledge/
    → injeta em .github/skills/bbds-api/SKILL.md
    → commit automático por bot do CI
```

**Workflow `bbds-api-sync.yml`:**
- Trigger: push em `package.json` do BBDS com version bump detectado
- Step 1: `npm install` do BBDS novo
- Step 2: `ts-node scripts/extract-bbds-types.ts`
- Step 3: Commit `bbds-api-reference.md` + skill atualizada
- Step 4: Abre PR de revisão (auto-merge após CI green)
- SLA: bbds bump → skill atualizada em <30min

### Estratégia de carregamento (três camadas)

```
copilot-instructions.md  → camada sempre carregada
    Contém: referência ao BBDS, comando para carregar skill bbds-api

Skills sob demanda       → carregadas por descrição no SKILL.md
    bbds-api:           "Referência de API dos componentes BBDS..."
    bbds-ux-guidelines: "Diretrizes de uso e composição UX para componentes BBDS..."
    bbds-patterns:      "Padrões de composição de telas React Native com BBDS..."

figma-to-code skill      → carregada por tarefa específica (skill existente)
    Aprimoramento: carrega bbds-api-reference antes de identificar componentes,
    cruza com API atual, usa migration paths para props @deprecated
```

### Impacto no figma-to-code skill (aprimoramento)

A skill existente `figma-to-code` deve ser aprimorada para:
1. Carregar `bbds-api-reference.md` antes de mapear componentes do Figma
2. Cruzar cada componente identificado contra a API atual (props existem? estão deprecated?)
3. Usar migration paths documentados para props obsoletas
4. Emitir warning se componente não existe no BBDS (potencial novo componente para o time)

### Evals da Fase 0b

- **Eval 1 — Acurácia de props** (code-based, pass@1): dado um uso de componente com prop errada, o agente corrige para a prop certa documentada no bbds-api-reference?
- **Eval 2 — Aderência UX** (model-based): o agente aplica `bbds-ux-guidelines.yaml` (avoid_when, anti_patterns)?
- **Eval 3 — figma-to-code cross-reference** (code-based): skill identifica props @deprecated e aplica migration path?
- **Eval 4 — Regressão em version bump** (code-based): após novo release do BBDS, evals anteriores ainda passam?

### Medição

| Métrica | Target |
|---|---|
| Tempo BBDS bump → skill atualizada | <30min (automatizado) |
| % PRs com uso incorreto de BBDS detectado em review | → 0% |
| Eval 1 pass@1 | >95% |
| Eval 3 pass@1 | >90% |

---

## Os 7 Agentes da Esteira

### Agente 01 — Intent Agent (Stage 1: Plan)

**O que muda:** Ideias saem do BusinessMap como artefato estruturado em minutos,
não dias de alinhamento em reuniões.

**Ferramenta**: Copilot Chat + BusinessMap MCP (VS Code agent mode)

**Input**: URL/ID do card no BusinessMap

**Output**: `intent.md` com: problema, outcome esperado, usuários afetados,
sistemas envolvidos, constraints, perguntas abertas, autor e timestamp.

**Fluxo concreto:**
1. Product Owner abre Copilot Chat no VS Code com BusinessMap MCP ativo
2. Prompt: `"Converta o card #[ID] do BusinessMap em um intent.md seguindo nosso template"`
3. Copilot chama o MCP, busca dados do card
4. Gera `intent.md` usando `templates/intent.md`
5. PO revisa, corrige, commita no repo do produto ou em repo de intents centralizado
6. Aprovação: merge = aceito; PR fechado sem merge = rejeitado

**Integração BusinessMap:**
- Fonte de verdade: BusinessMap (card permanece como authoritative)
- `intent.md` é cópia de trabalho vinculada ao card via link no frontmatter
- Commit no `intent.md` atualiza o card no BusinessMap via MCP (write-back)

**Evals do Agente 01:**
- `agents/01-intent/evals/tasks/`: 20-50 cards reais com intent.md esperado como ground truth
- Graders (code-based):
  - Todos os campos obrigatórios presentes? (regex/schema check)
  - Link para o card BusinessMap presente e válido?
  - Timestamp e autor preenchidos?
- Graders (model-based):
  - O problema declarado corresponde ao problema do card? (rubrica 1-5)
  - Os critérios de sucesso são mensuráveis?
  - As perguntas abertas são relevantes e não triviais?
- Métricas: pass@1 (geração única deve passar)

**Governança:** PO aprova via merge do PR. Versão do template de intent registrada.

**Medição:**
- Leading: Tempo do primeiro card → intent.md commitado (target: <4h)
- Lagging: Taxa de intent.md aceitos sem revisões maiores após o spec stage

---

### Agente 02 — Spec Agent (Stage 2: Design)

**Pré-requisito:** Fase 0b ativa — skills `bbds-ux-guidelines` e `bbds-patterns` disponíveis.
Sem elas, o agente especifica componentes sem critério de adequação ao design system.

**O que muda:** Requisitos e design colapsam em uma sessão. Políticas aplicadas
durante a geração, não descobertas em revisão semanas depois.

**Ferramenta**: Copilot agent mode + skills de política + skills BBDS (Fase 0b)

**Input**: `intent.md` aprovado + políticas da organização (via `.github/skills/`) + protótipo Figma (referência)

**Output**: `spec.md` com: requisitos funcionais, não-funcionais, constraints de design
(sem mockup — referência ao Figma), critérios de aceitação, flags de compliance,
**componentes BBDS recomendados por tela** (com justificativa via bbds-ux-guidelines).

**Nota sobre Design:** A fase de design visual continua manual (UX + engenheiros no
Figma). O spec.md documenta os requisitos de UX/design com referência ao arquivo
do Figma e mapeia quais componentes BBDS usar em cada tela. Quando o dev mode do Figma
estiver disponível em escala, o MCP do Figma pode ser ativado para puxar specs diretamente.

**Fluxo concreto:**
1. PO abre Copilot agent mode com `#file:intent.md`
2. Copilot carrega automaticamente as skills relevantes de `.github/skills/` (security,
   platform-standards, compliance, **bbds-ux-guidelines**, **bbds-patterns**) com base no contexto
3. Copilot gera spec.md sinalizando conflitos/riscos em seção `## Flags`,
   incluindo flag quando componente proposto viola `avoid_when` do bbds-ux-guidelines
4. PO endereça flags com policy owners antes de commitar
5. Commit de `spec.md` dispara workflow `intent-to-spec.yml` que valida campos

**GitHub Action `intent-to-spec.yml`:**
- Trigger: push de spec.md
- Valida schema do spec.md (campos obrigatórios, referência ao intent.md)
- Checa que todas as flags de compliance foram resolvidas ou documentadas
- Comenta no PR com score de completude

**Evals do Agente 02:**
- 20-50 pares (intent.md, spec.md esperado) de projetos reais
- Graders (code-based):
  - Todos os critérios de aceitação são verificáveis?
  - Seção `## Flags` existe e flags são categorizadas?
  - Referência ao `intent.md` está presente?
  - Componentes BBDS listados existem no `bbds-api-reference.md`? (sem componente inventado)
- Graders (model-based):
  - Spec endereça todos os pontos do intent?
  - Políticas de segurança foram aplicadas (não apenas mencionadas)?
  - Constraints de plataforma React Native estão presentes?
  - Componentes BBDS escolhidos respeitam `use_when`/`avoid_when` do bbds-ux-guidelines?
- Métricas: pass@1, taxa de flags resolvidas antes do merge

**Governança:** Skills em `.github/skills/` são version-controlled. Mudanças
requerem aprovação do policy owner. Versão das skills registrada no commit do spec.md.

**Medição:**
- Leading: Tempo entre intent.md e spec.md commitados (target: <1 dia)
- Lagging: Rework no spec após o build começar (commits de spec.md pós-plan.md)

---

### Agente 03 — Plan Agent (Stage 3a: Planning)

**O que muda:** Nenhuma implementação sem plano escrito aprovado. Conhecimento
institucional vira arquivo versionado no repo.

**Ferramenta**: Copilot agent mode + plan mode no VS Code

**Input**: `spec.md` + `.github/copilot-instructions.md` do repo-alvo

**Output**: `plan.md` com: arquivos que mudam, ordem de trabalho, riscos
identificados, estratégia de testes, critérios de "pronto".

**Pré-requisito por repo:** `.github/copilot-instructions.md` deve existir (gerado
na Fase 0 de bootstrap). Sem ele, o agente opera cego — bloquear via status check no CI.

**Fluxo concreto:**
1. Engenheiro abre VS Code no repo do bundle correto
2. Copilot agent mode com `#file:spec.md` + copilot-instructions.md carregado automaticamente
3. Prompt: `"Gere um plan.md para implementar esta spec no contexto deste repo"`
4. Copilot analisa o repo, lista arquivos afetados, propõe ordem de trabalho
5. Engenheiro interroga o plano (riscos, alternativas, impacto em outros bundles)
6. Itera até o plano estar implementável
7. Commita `plan.md` no repo
8. A implementação só começa após commit do plan.md

**Enforcement via GitHub Actions:**
```yaml
# .github/workflows/require-plan.yml
# Status check que bloqueia merge se plan.md não existe na branch
# Configurado como required check no branch protection do repo
```
Git hooks locais (pré-push via husky/lefthook) podem complementar para feedback imediato no dev.

**Evals do Agente 03:**
- Tasks: specs reais com plan.md esperado como ground truth
- Graders (code-based):
  - Todos os arquivos listados no plan existem no repo?
  - Seção de testes presente?
  - Riscos documentados?
  - Nenhum arquivo de segurança listado para alteração sem justificativa?
- Graders (model-based):
  - Plano é executável por um engenheiro novo no bundle?
  - Ordem de trabalho faz sentido técnico (dependências respeitadas)?
  - Escopo está dentro do spec (nem mais, nem menos)?
- Métricas: pass@1

**Medição:**
- Leading: Proporção de PRs com plan.md commitado (target: 100% após rollout)
- Lagging: Divergência entre plan.md e diff final do PR

---

### Agente 04 — Build Agent (Stage 3b: Implementation)

**Pré-requisito:** Fase 0b ativa — skill `bbds-api` e `figma-to-code` com acesso à `bbds-api-reference.md`.
Sem elas, o agente implementa componentes com props incorretas ou usa padrões obsoletos.

**O que muda:** Implementação guiada pelo plan.md aprovado. Padrões de plataforma
e API correta do BBDS aplicados automaticamente via skills.

**Ferramenta**: Copilot agent mode + Copilot Edits (VS Code) + skills BBDS (Fase 0b) + figma-to-code skill

**Input**: `plan.md` + `.github/copilot-instructions.md` + protótipo Figma referenciado no spec.md

**Output**: Código implementado + testes

**Fluxo concreto:**
1. Engenheiro abre Copilot Edits com `#file:plan.md` como contexto
2. Para telas: invoca `figma-to-code` skill (já aprimorada na Fase 0b para cruzar com bbds-api-reference)
3. Skill identifica componentes BBDS no protótipo Figma, valida props contra API atual, aplica migration paths
4. Implementação em ordem definida no plan.md; skill `bbds-api` carregada automaticamente
5. Copilot verifica contra platform-standards + BBDS automaticamente (via copilot-instructions.md + skills)
6. Ao final: `npm test` / `jest` — loop de feedback antes de push
7. Se plan.md precisar ser alterado: atualizar o arquivo e justificar no commit

**Loop de feedback:**
- Comando de verificação documentado no `copilot-instructions.md` do repo
- Copilot executa e lê resultado, corrige antes do push humano ver

**Evals do Agente 04:**
- Tasks: plan.md + repo state → código esperado
- Graders (code-based):
  - Testes passam? (binary)
  - Lint/format pass?
  - Nenhum `console.log` ou debug code no diff?
  - Platform naming conventions seguidas? (regex)
  - Imports de bibliotecas não-aprovadas? (allowlist check)
  - Props BBDS usadas existem em `bbds-api-reference.md`? (sem prop inventada)
  - Props `@deprecated` usadas sem migration path aplicado? (deve ser 0)
- Graders (model-based):
  - Implementação é fiel ao plan.md?
  - Novos componentes seguem padrões do design system (bbds-patterns)?
- Métricas: pass@1 (CI first-pass), pass^3 (consistência)

**Governança:** Platform standards em copilot-instructions.md são versionados e
aprovados pela equipe de plataforma. Build agent não pode editar arquivos de teste
durante uma run de fix — enforçado via instrução explícita no copilot-instructions.md
e verificado por status check no GitHub Actions.

**Medição:**
- Leading: Taxa de CI first-pass em mudanças geradas com agente (target: >80%)
- Lagging: Ciclos de rework por PR vs. média histórica

---

### Agente 05 — Test Agent (Stage 4: Test)

**O que muda:** Configuração do agente (copilot-instructions.md, skills) é testada
como código. Regressões detectadas antes de chegar ao engenheiro.

**Ferramenta**: GitHub Actions + Copilot API (para análise de falhas)

**Input**: Code diff + resultados de CI + histórico de evals

**Output**: Relatório de testes, sugestões de correção, eval pass rate

**Fluxo concreto:**
1. Push dispara GitHub Action com suite de testes do repo
2. Para falhas de CI: Copilot analisa o erro e sugere correção em comentário no PR
3. Eval suite roda em paralelo:
   - Tasks de `agents/05-test/evals/tasks/` executadas no modo non-interactive
   - Graders aplicados
   - Pass rate calculado
4. Se pass rate < threshold: PR bloqueado (branch protection rule)
5. Quando incidente de produção ocorre: novo eval task criado a partir do bug

**GitHub Action `eval-suite.yml`:**
- Trigger: push para main/staging ou mudança em `.github/copilot-instructions.md` / `.github/skills/`
- Roda todas as tasks de eval de todos os 7 agentes
- Report de pass rate por agente
- Gera badge e histórico de tendência

**Evals do Agente 05:**
- Meta-eval: O Test Agent identifica falhas reais e não falsos positivos?
- Tasks: logs de CI com falhas reais vs. diagnóstico esperado
- Graders:
  - Diagnóstico de falha é correto? (model-based com ground truth)
  - Sugestão de fix resolve o problema? (code-based: apply fix, run tests)
  - Falsos positivos: agent reporta erro quando não há erro? (deve ser 0)

**Governança:** Threshold de pass rate definido em `eval-suite.yml` (ex: 85%).
Mudança no threshold requer PR aprovado pelo tech lead. Incidentes de produção
viram evals permanentes — "não pode acontecer de novo".

**Medição:**
- Leading: Eval pass rate over time (KPI semanal)
- Lagging: Regressões capturadas em CI vs. chegadas em produção

---

### Agente 06 — Deploy Agent (Stage 5: Deploy)

**O que muda:** Review corre bidirecionalmente. Copilot revisa PRs incoming e endereça
comentários de review. Humano aprova, não substitui.

**Ferramenta**: Copilot Code Review (Enterprise) + GitHub Actions

**Input**: PR diff + `REVIEW.md` + `spec.md` + `plan.md`

**Output**: Review comments (bugs, segurança, compliance), aprovação/rejeição de CI

**Estrutura do `REVIEW.md`:**
```markdown
## Passes de Review
1. Bugs e erros de lógica (severidade: Important)
2. Segurança / vulnerabilidades (severidade: Important)
3. Compliance com spec.md e plan.md (severidade: Important)
4. Padrões de plataforma React Native (severidade: Nit se cosmético)

## Exclusões
- Arquivos gerados (*.generated.ts, __mocks__)
- Regras já enforçadas por lint/CI

## Limites
- Máximo 10 nits por PR
- Important findings: ilimitado
```

**Fluxo concreto:**
1. PR aberto → Copilot Code Review roda automaticamente (Enterprise feature)
2. Review usa `REVIEW.md` como rubrica, `spec.md` como spec de referência
3. Engenheiro menciona `@copilot` em comentários para que ele endereça
4. Branch protection: requer 1 humano + CI green para merge
5. Agente não pode aprovar o próprio código (separação de funções)
6. Achados de review alimentam `copilot-instructions.md` (ciclo de aprendizado)

**Gates de aprovação:**
```yaml
# Prod gate via GitHub Actions environment protection
# Deploy para prod requer aprovação manual de release manager
# Configurado como "required reviewers" no environment "production"
```

**Tiering de autonomia:**
- Dev: agente deploya livremente (via workflow)
- Staging: agente abre PR, CI aprova, deploy automático
- Prod: agente prepara, release manager autoriza via GitHub environment approval

**Evals do Agente 06:**
- Tasks: PRs reais com review esperado como ground truth
- Graders (code-based):
  - Todos os bugs reais do PR foram sinalizados?
  - Falsos positivos (findings inválidos) < 10%?
  - Findings categorizados corretamente (Important vs. Nit)?
- Graders (model-based):
  - Finding tem evidência clara (linha + explicação)?
  - Sugestão de fix é correta?
- Métricas: precision e recall de findings reais

**Medição:**
- Leading: Tempo até primeiro review (target: <15min após PR aberto)
- Lagging: Defeitos/vulnerabilidades capturados pre-merge vs. pós-deploy; DORA metrics

---

### Agente 07 — Maintain Agent (Stage 6: Maintain)

**O que muda:** Três camadas de monitoramento autônomo — técnica (crashes), negocial
(jornadas) e qualidade (CI) — convergem num pipeline de correlação que o Copilot analisa
antes de gerar o `intent.md`. Achados re-entram no pipeline formalmente. Loop se auto-perpetua.

**Soluções existentes (a preservar e estender):**
- Cron job no GitHub Actions com Firebase MCP: top crashes → desminificação → root cause → GitHub issue
- Ferramenta de monitoramento de jornadas: detecta queda de taxa de sucesso por fluxo negocial e
  identifica a **tela problemática** na jornada; inclui detecção de erros de endpoint por fluxo

**Ferramentas**: GitHub Actions (cron agendado) + Firebase MCP + Journey Monitor API/MCP + Copilot API

**Três fontes de monitoramento:**

| Fonte | O que detecta | Localização entregue | Path de análise |
|---|---|---|---|
| Firebase Crashlytics | Falha técnica (crash) | Stacktrace → arquivo:linha | Direto: desminifica → root cause |
| Ferramenta de jornada | Falha negocial (usuário não completou) | Flow + tela problemática | Correlação: crash? endpoint? commit recente? |
| bands.yaml (CI/ops) | Degradação de qualidade | Métricas de CI, cycle time | Determinístico: Western Electric rules |

**Input**: Sinais de todas as três fontes, processados em sequência por um pipeline de correlação

**Output para todas as fontes:**
- GitHub issue (visibilidade imediata para on-call)
- PR com `intent.md` estruturado (entrada formal no pipeline SDLC)

---

#### Pipeline de Correlação (novo — central para falhas de jornada)

Falhas de jornada têm múltiplas causas possíveis. Antes do Copilot analisar, uma etapa
determinística de correlação reúne todos os sinais relevantes para a tela problemática:

```
[Journey Monitor] detecta queda de taxa de sucesso
    → entrega: flow_id, screen_id, taxa de queda (%), endpoint_errors?

GitHub Actions — steps paralelos de correlação:
    ├── Firebase MCP: há crash na screen_id no mesmo período? → sim/não
    ├── git log --since=48h -- *<screen_id>*: commit recente toca essa tela? → sim/sha/não
    └── gh release list: deploy recente cobrindo esse bundle? → sim/versão/não

Copilot recebe o "correlation bundle" e segue árvore de diagnóstico:

    IF endpoint_error → problema de backend
        action: ping time de API, avaliar rollback
        route: intent.md para time responsável pelo endpoint

    ELIF crash_found → bug técnico (caminho Firebase)
        action: stacktrace já desminificado → root cause
        route: intent.md com stacktrace incluso

    ELIF commit_recent → regressão provável
        action: bisect candidate, avaliar rollback do commit
        route: intent.md com sha do commit e arquivos afetados

    ELSE → bug de lógica/UX (silent failure)
        action: investigação manual com session recording
        route: intent.md com hipótese aberta, sinais coletados como evidência
```

**Por que a árvore é determinística:** garante que o Copilot não "invente" hipóteses —
ele segue o caminho que os dados indicam. Isso também torna cada branch testável
em evals com casos conhecidos.

---

#### Configuração `bands.yaml` (três fontes)

```yaml
# Métricas de CI/operação
- metric: ci_test_failure_rate
  baseline: rolling_30d
  tiers:
    1sigma: { action: log }
    2sigma: { action: diagnose, tools: "read,grep,gh-run-view" }
    3sigma: { action: propose, routes: [pull_request, runbook:rollback-deploy] }

# Crash rate geral (Firebase)
- metric: firebase_crash_rate
  baseline: rolling_7d
  source: firebase_mcp
  tiers:
    1sigma: { action: log }
    2sigma: { action: diagnose, tools: "firebase-mcp,stacktrace-deobfuscate,gh-issue" }
    3sigma: { action: propose, routes: [intent_md, runbook:rollback-deploy] }

# Novo crash class (Firebase) — sem baseline, qualquer ocorrência = ação
- metric: firebase_new_crash_class
  baseline: none
  source: firebase_mcp
  tiers:
    any: { action: diagnose, tools: "firebase-mcp,stacktrace-deobfuscate,gh-issue,intent_md" }

# Taxa de sucesso de jornada por fluxo negocial (Journey Monitor)
- metric: journey_success_rate
  by: [flow_id, screen_id]     # granularidade até a tela problemática
  baseline: rolling_7d
  source: journey_monitor
  tiers:
    1sigma: { action: log }
    2sigma: { action: correlate, tools: "firebase-mcp,git-log,gh-release,journey-monitor" }
    3sigma: { action: diagnose+intent_md, uses: correlation_pipeline }
```

---

#### Fluxo concreto — crashes (Firebase, cron existente + extensão)

1. Cron roda; Firebase MCP busca top crashes + novos crash classes
2. Para cada crash relevante: desminificação → análise Copilot → hipótese de root cause
3. GitHub issue aberta (comportamento atual) + PR com `intent.md` (extensão)
4. On-call: merge PR = entra no pipeline; close = dismiss (tunea sensibilidade dos bands)
5. Fix via pipeline normal; incidente vira eval permanente no Agente 05

#### Fluxo concreto — jornadas (Journey Monitor + correlação)

1. Cron consulta Journey Monitor API/MCP; detecta flows com queda de taxa de sucesso
2. Para cada flow afetado: obtém `flow_id`, `screen_id`, `endpoint_errors`
3. Correlação paralela: Firebase MCP (crash na tela?) + git log (commit recente?) + gh release (deploy?)
4. Copilot recebe correlation bundle → aplica árvore de diagnóstico → hipótese + action
5. GitHub issue aberta + PR com `intent.md` estruturado (exemplo abaixo)
6. On-call triagem: merge ou dismiss; dismiss por "known issue" também tunea os bands

**Exemplo de `intent.md` gerado por falha de jornada:**
```markdown
---
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
  Causa provável: regressão na lógica de validação de crédito (commit abc123).
action: ping time de API + avaliar rollback de v2.4.1
---
```

#### Fluxo concreto — métricas de CI (bands.yaml)

1. Script determinístico (Western Electric rules) detecta breach
2. GitHub Action dispara Copilot em modo non-interactive
3. Copilot analisa em read-only → `intent.md` com diagnóstico
4. Mesmo ciclo de triagem via PR

**Scans de segurança:**
- GitHub Advanced Security (CodeQL, Dependabot) como base determinística
- Copilot para análise de achados complexos e geração de patches
- Achados pontuais: PR de patch direto; achados sistêmicos: `intent.md` + pipeline completo

---

**Evals do Agente 07:**
- Tasks: histórico de incidentes reais com diagnóstico correto como ground truth
- **Tasks de crash analysis (Firebase):**
  - Stacktrace desminificado → root cause esperado (ground truth: fix que veio depois)
  - Novo crash class → intent.md com campos obrigatórios e problema descrito?
  - Crash known → agente identifica como recorrente, não abre duplicate?
- **Tasks de correlação (Journey Monitor) — uma por branch da árvore:**
  - Jornada com endpoint_error → diagnosis_path correto? action correto?
  - Jornada com crash correlacionado → diagnóstico usa o stacktrace?
  - Jornada com commit recente, sem crash, sem endpoint → regressão identificada?
  - Jornada sem correlação → agente não fabrica hipótese, mantém aberta?
- Graders (code-based):
  - `intent.md` tem todos os campos obrigatórios (source, flow, screen, correlation, hypothesis)?
  - `diagnosis_path` é um dos valores válidos da árvore?
  - Duplicate detection: não abre nova issue/intent para problema já aberto e não resolvido?
  - Tempo de geração do intent.md < SLA (30min)?
- Graders (model-based):
  - Hypothesis é plausível dado o correlation bundle?
  - Action é proporcional à severity e ao diagnosis_path?
- Métricas: time-to-intent, diagnosis_path accuracy, resolution rate, repeat incident rate

**Medição:**
- Leading: Tempo de detecção → intent.md na fila de triagem (target: <30min)
- Lagging: % de achados que viram fixes; incidentes repetidos por classe (-50% em 6m);
  diagnosis_path correto vs. root cause real confirmado no post-mortem

---

## Legacy / Repos Existentes — Estratégia de Migração

**Contexto:** Dezenas de repos React Native sem `copilot-instructions.md`, sem
artefatos MD, sem evals. Cada bundle é independente.

**Estratégia: Bootstrap Progressivo**

### Nível 0 — Repo sem nenhum contexto (situação atual)
Copilot opera sem contexto de plataforma → alto risco de código fora do padrão.

### Nível 1 — Bootstrap mínimo (target para todos os repos em 60 dias)
Arquivo `.github/copilot-instructions.md` com:
- Contexto do bundle (nome, responsabilidade, dependências)
- Comandos de build/test/lint
- Padrões críticos (anti-patterns conhecidos deste repo)
- Referência ao template de platform-standards

**Como fazer:** GitHub Action `bootstrap-repo.yml` abre PR em cada repo com:
1. `.github/copilot-instructions.md` gerado pelo Copilot (analisa o repo existente)
2. Templates vazios de `intent.md`, `spec.md`, `plan.md`
3. `REVIEW.md` com critérios de revisão base
4. Checklist de validação para o time do bundle

### Nível 2 — Evals mínimos (target para repos ativos em 90 dias)
20 tasks de eval baseadas nas tarefas mais comuns do bundle.

### Nível 3 — Pipeline completo (target para novos fluxos)
Todos os 7 agentes ativos, metrificados, com evals cobrindo 80% dos casos de uso.

**Priorização:** Repos com maior volume de mudanças/semana primeiro.

---

## Skills — Fase 2 (Registrado para trabalho posterior)

As seguintes skills precisam ser criadas em um segundo momento, com auxílio na
definição das políticas. **Formato:** SKILL.md com frontmatter YAML `name` e
`description` — compatível com GitHub Copilot e Claude Code.

| Skill | Arquivo | Conteúdo | Responsável |
|---|---|---|---|
| `branding` | `.github/skills/branding/SKILL.md` | Cores, tipografia, voz/tom, identidade visual | Marketing/Design |
| `security` | `.github/skills/security/SKILL.md` | OWASP RN, secure storage, cert pinning, data protection | Security team |
| `compliance` | `.github/skills/compliance/SKILL.md` | LGPD/GDPR, WCAG 2.1, data residency | Legal/Compliance |
| `ux` | `.github/skills/ux/SKILL.md` | Design system components, interaction patterns, navigation | UX team |

**Nota:** O Copilot carrega skills automaticamente com base no campo `description` do SKILL.md,
sem necessidade de referência explícita no prompt. Escrever a `description` com precisão é crítico.

**Para cada skill, o processo será:**
1. Identificar o policy owner e a fonte de verdade atual
2. Extrair e formalizar em `.github/skills/<nome>/SKILL.md`
3. Definir `description` precisa (é o trigger que faz o Copilot carregar a skill)
4. Testar com 10 exemplos positivos + 10 negativos
5. Validar com o policy owner via PR
6. Adicionar eval de regressão no suite do Agente 02 (Spec) e Agente 04 (Build)

---

## Governança — Cadeia de Aprovação

```
intent.md  →  [Product Owner: merge PR]
    ↓
spec.md    →  [Product Owner: merge PR] + [Policy owners: flags resolvidas]
    ↓
plan.md    →  [Engenheiro: commit + code review pass]
    ↓
Código     →  [Copilot Code Review] + [1 humano: branch protection]
    ↓
Deploy     →  [CI green] + [Staging: automático] + [Prod: release manager]
    ↓
Incidente  →  [intent.md automático] + [On-call: triagem]
```

**Separação de funções:**
- Agente não aprova o próprio código
- Prod gate requer humano nomeado
- Policy changes requerem policy owner
- Todos os agentes logados com identidade de CI (não eng individual)

---

## Medição — KPIs por Stage

| Stage | Leading Indicator | Target | Lagging Indicator | Target |
|---|---|---|---|---|
| 1 Plan | Tempo card → intent.md | <4h | Taxa intent aceitos sem rework | >85% |
| 2 Design | Tempo intent → spec | <1 dia | Rework de spec após build | <10% |
| 3 Build | % PRs com plan.md | 100% | Divergência plan vs. diff | <20% |
| 4 Test | Eval pass rate | >85% | Regressões em CI vs. prod | >90% em CI |
| 5 Deploy | Tempo ao primeiro review | <15min | Defect catch pre-merge | >80% |
| 6 Maintain | Tempo breach → intent.md | <30min | Repeat incidents / classe | -50% em 6m |

---

## Verificação — Como saber se funcionou

### Por stage:
1. **Intent**: Abrir card real no BusinessMap, gerar intent.md, verificar campos e link
2. **Spec**: Partir de um intent.md existente, gerar spec.md, checar flags e compliance
3. **Plan**: Gerar plan.md para uma spec real, verificar arquivos listados existem no repo
4. **Build**: Implementar plan.md, rodar `npm test`, checar CI green first-pass
5. **Test**: Fazer mudança intencional que quebra padrão → eval suite captura?
6. **Deploy**: Abrir PR com bug conhecido → Copilot Code Review identifica?
7. **Maintain**: Simular breach de métrica → intent.md gerado em <30min?

### Suite de evals end-to-end:
- `eval-suite.yml` roda todos os 7 agentes em sequência com inputs reais
- Pass rate geral ≥ 85% = esteira funcional
- Trends semanais plotados em dashboard (GitHub Actions summary page ou similar)

---

## Sequência de Implementação Recomendada

```
Semana 1-2:  Fase 0 — Bootstrap de conhecimento de plataforma
Semana 2-3:  Fase 0b — Arquitetura de conhecimento BBDS
               → scripts/extract-bbds-types.ts + bbds-api-sync.yml
               → bbds-api-reference.md gerado + bbds-ux-guidelines.yaml (UX)
               → três skills BBDS ativas + figma-to-code aprimorada
               → evals Fase 0b validados
Semana 3-4:  Templates + Agente 01 (Intent) + evals iniciais
Semana 4-5:  Agente 02 (Spec) + skills de policy (security + platform-standards + BBDS)
Semana 5-6:  Agente 03 (Plan) + GitHub Actions de enforcement + copilot-instructions base
Semana 6-7:  Agente 04 (Build) + feedback loop + figma-to-code integrado
Semana 7-8:  Agente 05 (Test) + eval suite em CI
Semana 8-9:  Agente 06 (Deploy) + Copilot Code Review + REVIEW.md
Semana 9-11: Agente 07 (Maintain) + bands.yaml + monitor workflow
Semana 11+:  Bootstrap de repos existentes (progressivo)
Fase 2 TBD:  Skills de branding, compliance, ux (criação com policy owners)
```

---

## Decisões Abertas / Próximos Passos

1. **CI/CD híbrido**: Identificar qual parte do pipeline usa GitHub Actions vs. outro sistema
   para mapear onde cada workflow vai rodar.

2. **Eval runner**: Confirmar que GitHub Actions tem acesso à Copilot API em modo
   non-interactive para rodar evals automatizados.

3. **BusinessMap write-back**: Confirmar se o MCP do BusinessMap suporta write
   (atualizar card com link para intent.md) além de read.

4. **Figma — longo prazo**: Quando licenças dev mode estiverem disponíveis em escala,
   adicionar Figma MCP ao Agente 02 (Spec) para puxar specs diretamente.

5. **Skills — Fase 2**: Agendar sessões de trabalho com policy owners (branding,
   security, compliance, UX) para criar as 4 skills pendentes.
