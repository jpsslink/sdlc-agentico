# Inventário de Conhecimento de Plataforma

**Status:** Em construção — Fase 0-A (Inventário) não iniciada.

Este documento é produzido durante a Fase 0-A da esteira agêntica. Ver [ADR-012](decisions.md#adr-012--auditoria-e-validação-da-base-de-conhecimento-como-pré-requisito-da-fase-0) para o processo completo.

---

## Como usar este documento

Para cada domínio de conhecimento:
- **Fontes existentes:** onde o conhecimento existe hoje (repo, wiki, Figma, conhecimento tácito)
- **Status:** `ok` (validado por SME) | `desatualizado` | `incompleto` | `gap` (não existe) | `pendente-validação`
- **SME responsável:** quem valida e mantém esse domínio
- **Skill alvo:** qual skill vai encodar esse conhecimento
- **Processo de manutenção:** como a skill é atualizada quando o conhecimento muda

Skills com status diferente de `ok` **não entram em produção**.

---

## Domínios de Conhecimento

### Arquitetura de Bundle

| Campo | Valor |
|---|---|
| **Fontes existentes** | — |
| **Status** | `pendente-validação` |
| **SME responsável** | — |
| **Skill alvo** | `platform-standards` |
| **Processo de manutenção** | — |

**Gaps identificados:**
- _a preencher na Fase 0-A_

---

### Convenções de Código

| Campo | Valor |
|---|---|
| **Fontes existentes** | — |
| **Status** | `pendente-validação` |
| **SME responsável** | — |
| **Skill alvo** | `platform-standards` |
| **Processo de manutenção** | — |

**Gaps identificados:**
- _a preencher na Fase 0-A_

---

### Anti-patterns Documentados

| Campo | Valor |
|---|---|
| **Fontes existentes** | — |
| **Status** | `pendente-validação` |
| **SME responsável** | — |
| **Skill alvo** | `platform-standards` |
| **Processo de manutenção** | — |

**Gaps identificados:**
- _a preencher na Fase 0-A_

---

### Segurança — OWASP React Native

| Campo | Valor |
|---|---|
| **Fontes existentes** | — |
| **Status** | `pendente-validação` |
| **SME responsável** | — |
| **Skill alvo** | `security` |
| **Processo de manutenção** | — |

**Gaps identificados:**
- _a preencher na Fase 0-A_

---

### Segurança — Secure Storage, Cert Pinning, Autenticação

| Campo | Valor |
|---|---|
| **Fontes existentes** | — |
| **Status** | `pendente-validação` |
| **SME responsável** | — |
| **Skill alvo** | `security` |
| **Processo de manutenção** | — |

**Gaps identificados:**
- _a preencher na Fase 0-A_

---

### BBDS — API de Componentes

| Campo | Valor |
|---|---|
| **Fontes existentes** | TypeScript types no pacote `@bbds/components` |
| **Status** | `pendente-validação` |
| **SME responsável** | Time BBDS |
| **Skill alvo** | `bbds-api` (auto-gerada via ts-morph — ver ADR-006) |
| **Processo de manutenção** | Auto-geração em CI após cada release do BBDS |

**Gaps identificados:**
- Pipeline de geração via ts-morph não implementado
- _outros gaps a preencher na Fase 0-A_

---

### BBDS — Guidelines de UX

| Campo | Valor |
|---|---|
| **Fontes existentes** | — |
| **Status** | `pendente-validação` |
| **SME responsável** | Time de UX |
| **Skill alvo** | `bbds-ux-guidelines` |
| **Processo de manutenção** | Curação manual — revisão a cada release com mudança de componente |

**Gaps identificados:**
- _a preencher na Fase 0-A_

---

### BBDS — Padrões de Composição de Telas

| Campo | Valor |
|---|---|
| **Fontes existentes** | — |
| **Status** | `pendente-validação` |
| **SME responsável** | — |
| **Skill alvo** | `bbds-patterns` |
| **Processo de manutenção** | — |

**Gaps identificados:**
- _a preencher na Fase 0-A_

---

## Domínios Fora de Escopo (Fase 2)

Os domínios abaixo estão planejados para Fase 2 e não fazem parte da auditoria inicial:
- `branding` — identidade visual e tom de comunicação
- `compliance` — LGPD, WCAG, data residency
- `ux` — padrões de navegação e interação além do BBDS

---

## Histórico de Atualizações

| Data | Etapa | Responsável | O que mudou |
|---|---|---|---|
| — | Fase 0-A | — | Documento criado como placeholder |
