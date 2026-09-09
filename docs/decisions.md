# Decisões de Arquitetura

Este documento registra as decisões significativas por trás da esteira, com o contexto que as motivou, as alternativas que foram consideradas e as consequências de cada escolha. O objetivo é que qualquer membro do time consiga entender o *porquê* de cada decisão — não apenas o *o quê* — mesmo sem ter acompanhado as discussões originais.

Formato: **Contexto → Decisão → Justificativa → Alternativas Consideradas → Consequências**

---

## ADR-001 — GitHub Copilot Enterprise como Runtime

### Contexto

Para implementar a esteira, é necessário um **runtime de IA** — o sistema que executa os agentes. Esse runtime precisa: carregar contexto de plataforma de forma dinâmica, acessar ferramentas externas (como o BusinessMap MCP), operar em modo agêntico (planejar antes de executar), revisar código automaticamente, e ser acessível a todos os times sem onboarding adicional.

Existem múltiplas opções de runtime disponíveis no mercado.

### Decisão

GitHub Copilot Enterprise é o runtime exclusivo de todos os agentes da esteira.

### Justificativa

**Já adotado na organização.** O Copilot Enterprise está padronizado. Nenhum novo contrato, sem processo de aprovação adicional, sem curva de aprendizado para os times — o IDE já tem o Copilot instalado. Introduzir uma segunda ferramenta de IA fragmentaria o suporte e geraria resistência de adoção.

**Todas as capacidades necessárias existem nativamente:**
- Plan Mode: o agente escreve um plano antes de executar, visível para o engenheiro aprovar
- `runSubAgent`: capacidade de delegar subtarefas para outro agente especializado
- Skills no formato SKILL.md: mecanismo de carregamento dinâmico de contexto
- Copilot Code Review: revisão automática de PRs com rubrica configurável (`REVIEW.md`)
- Integração nativa com GitHub Actions: sem glue code adicional para triggers de CI

**Formato de skills compatível.** SKILL.md funciona identicamente em Copilot Enterprise e Claude Code (Anthropic). Se a organização mudar de runtime no futuro, as skills, as instruções e toda a infraestrutura de conhecimento não precisam ser reescritas. O investimento é portável.

**Sem fragmentação de acesso.** Uma ferramenta de IA para todos os times, com políticas centralizadas via GitHub Copilot Enterprise settings. Sem necessidade de gerenciar API keys individuais ou acessos paralelos.

### Alternativas Consideradas

**Claude Code (Anthropic):** Capacidades equivalentes — o playbook que inspira esta esteira foi desenvolvido com Claude Code. Mas requer onboarding adicional para todos os times e não está padronizado na organização. Seria um segundo runtime paralelo com custo de gestão adicional.

**GitHub Actions + LLM via API:** Possível tecnicamente, mas perde a experiência integrada ao IDE (onde o desenvolvedor já está). Exige infraestrutura própria (API calls, gestão de prompts, logging), sem a experiência interativa do Plan Mode. Mais adequado para automações de CI puras que não envolvem o desenvolvedor em tempo real.

### Consequências

- A esteira funciona dentro do fluxo de trabalho existente do desenvolvedor — no IDE, sem nova ferramenta
- Mudanças nas capacidades do Copilot Enterprise (novos features, limitações) afetam diretamente a esteira
- A opção de migrar para Claude Code no futuro permanece aberta — skills são portáveis

---

## ADR-002 — Injeção Determinística de Conhecimento em vez de RAG

**Escopo desta ADR:** decide o **princípio de qualidade** do conhecimento injetado nos agentes — determinístico e curado vs. recuperação probabilística (RAG). Não decide o formato concreto (SKILL.md, endpoint MCP, pacote npm) nem o mecanismo de distribuição — ambos são consequência da decisão em aberto em [ADR-009](#adr-009--arquitetura-de-distribuição-de-conhecimento-decisão-em-aberto).

### Contexto

Os agentes precisam de contexto de domínio: padrões de plataforma, API do design system, guidelines de UX, regras de segurança. Esse conhecimento existe hoje de forma dispersa e inacessível — em Confluence, em PRs antigos, na memória de engenheiros sênior.

Há duas abordagens arquiteturais para injetar esse conhecimento em um agente:

**Opção A — RAG (Retrieval-Augmented Generation):** indexar toda a documentação disponível em um banco de dados de vetores e recuperar os fragmentos mais relevantes a cada consulta. O agente faz uma pergunta; o sistema busca os chunks mais próximos semanticamente e os inclui no contexto.

**Opção B — Injeção determinística de conhecimento curado:** escrever documentação curada por domínio (API do BBDS, padrões de segurança, etc.), versionada e com curadoria humana ou auto-geração, e injetá-la nos agentes de forma previsível e completa — sem retrieval probabilístico. O formato concreto (arquivo SKILL.md por repo, endpoint de MCP server central, pacote versionado) depende da arquitetura de distribuição escolhida.

### Decisão

O conhecimento de domínio é injetado nos agentes de forma **determinística e curada** — não via RAG sobre documentações indexadas.

Esta decisão se aplica independentemente do formato ou mecanismo de entrega. Nas três alternativas de distribuição avaliadas em [ADR-009](#adr-009--arquitetura-de-distribuição-de-conhecimento-decisão-em-aberto), o princípio se mantém:

| Alternativa de distribuição | Como o determinismo se manifesta |
|---|---|
| MCP server central (Stripe model) | Endpoints estruturados servidos de forma previsível — nenhum retrieval semântico |
| Pacote npm versionado (Shopify model) | SKILL.md bundled em pacote — versão pinada, conteúdo sempre o mesmo |
| Arquivos per-repo (modelo padrão Copilot) | SKILL.md em cada repositório — carregado inteiro, sem retrieval |

O **formato concreto** (SKILL.md, MCP endpoint, pacote) **não está decidido aqui** — é consequência da arquitetura de distribuição definida em ADR-009.

### Justificativa

**RAG introduz incerteza onde precisamos de determinismo:**

RAG depende de recuperação probabilística — o chunk certo pode não ser retornado dependendo de como a consulta foi formulada ou como os embeddings foram indexados. Para contexto de segurança e compliance ("quais dados não podem ser logados?"), "às vezes carrega a regra certa" não é aceitável. Um agente que segue inconsistentemente as regras de segurança é mais perigoso do que um que as segue de forma previsível.

RAG também requer infraestrutura adicional que não existe hoje: banco de dados de vetores, pipeline de embedding com curadoria de qualidade dos chunks, tuning contínuo de retrieval e monitoramento de qualidade. Isso é investimento significativo antes de qualquer agente rodar.

**Injeção determinística é auditável e testável em evals:**

Com injeção determinística, o conteúdo exato que o agente recebe em cada situação é conhecido e estável. Mudanças no conhecimento passam por PR com aprovação do policy owner (para segurança e compliance) — há trilha de auditoria completa. Mais criticamente: **evals são reproduzíveis**. Um eval que testa se o agente aplica a regra de segurança X pode ser executado de forma idêntica em qualquer momento porque o contexto injetado não muda entre execuções. Com RAG, o contexto varia a cada run e os evals perdem confiabilidade.

**O volume de conhecimento de plataforma não justifica RAG:**

O conhecimento de plataforma é um conjunto bem delimitado de padrões específicos: a API de ~50-100 componentes BBDS, padrões de segurança para mobile, convenções de alguns bundles. Esse volume cabe em documentação curada de tamanho razoável. RAG seria overhead de infraestrutura para um problema que injeção direta resolve.

**Auto-geração resolve o drift de documentação sem RAG:**

O argumento para RAG costuma ser que documentação manual fica desatualizada. A resposta não é RAG — é auto-geração: a documentação da API do BBDS pode ser gerada via ts-morph a partir dos TypeScript types, nunca ficando desatualizada porque não depende de escrita manual.

### Alternativas de Formato Consideradas

As alternativas abaixo foram descartadas para o problema de conhecimento de plataforma curado. Para outros casos de uso (ex: busca em grandes corpora não-estruturados), podem ser válidas:

**RAG sobre Confluence/Notion:** Depende da qualidade da documentação existente (baixa e desatualizada). Adiciona infraestrutura de embedding + retrieval. O problema de lag de documentação manual persiste. Evals não são reproduzíveis.

**Few-shot examples no prompt:** Funciona para padrões simples e estáticos, mas não escala. Não é versionável separadamente do prompt principal.

**Fine-tuning:** Caro, requer dados de treinamento que não existem, e o conhecimento fica "congelado" no modelo — cada mudança de padrão exigiria re-treinamento. Inviável para ambiente com releases frequentes.

### Consequências

- O conhecimento de plataforma precisa ser **curado ativamente** — para domínios estáticos (segurança, padrões), curadoria humana via PR; para domínios dinâmicos (API do BBDS), auto-geração em CI
- O conteúdo injetado é auditável: qualquer comportamento inesperado do agente pode ser rastreado até o conhecimento que recebeu em uma sessão específica
- Evals são determinísticos e reproduzíveis — o mesmo input produz o mesmo contexto injetado em qualquer run
- Novos domínios de conhecimento requerem curadoria deliberada com aprovação do policy owner — não acontece automaticamente
- **O formato concreto e o mecanismo de distribuição estão em aberto** — decididos em [ADR-009](#adr-009--arquitetura-de-distribuição-de-conhecimento-decisão-em-aberto) e detalhados em [`docs/knowledge-governance.md`](knowledge-governance.md)

---

## ADR-003 — Cadeia de Artefatos Explícita (intent → spec → plan)

### Contexto

O ciclo atual de desenvolvimento vai de "card no BusinessMap" a "PR aberto" sem artefatos formais intermediários. O intent do produto fica em comentários de Slack. A spec fica em conversas entre PO e engenheiro. O plan de implementação existe na cabeça do desenvolvedor. O código não rastreia de volta ao problema original.

As consequências são concretas: incidentes de produção sem contexto para diagnóstico, code reviews que descobrem mal-entendimento de requisitos após a implementação completa, e decisões técnicas que não podem ser auditadas porque nunca foram registradas.

### Decisão

Cada estágio do SDLC produz um artefato Markdown versionado em git. Nenhum estágio começa sem o artefato anterior aprovado via merge de PR.

### Justificativa

Esta decisão vem diretamente do [playbook da Anthropic](https://claude.com/blog/the-ai-native-sdlc-playbook) e resolve problemas concretos do ciclo atual:

**Rastro auditável completo.** Qualquer linha de código pode ser rastreada até o `intent.md` — e até o card do BusinessMap. "Por que esse código existe?" tem resposta objetiva. "O que esse código deveria fazer?" tem resposta no `spec.md`. "Como foi decidido implementar assim?" está no `plan.md`.

**Prevenção de drift entre intenção e implementação.** O Agente 04 (Build) lê o `plan.md` e implementa item por item. O Agente 06 (Deploy) lê o `spec.md` e verifica se o código implementa os critérios de aceitação. Se o código divergir do plano, o agente detecta e sinaliza — em vez de o revisor humano descobrir no final do ciclo.

**Decisões conscientes nos gates.** A aprovação de cada artefato é um gate humano explícito — o PO que aprova o `intent.md` está confirmando que o problema está correto. O engenheiro que commita o `plan.md` está confirmando que a estratégia é viável. Não é um "lgtm" rápido num PR com 500 linhas de código.

**Contexto preservado para incidentes.** Quando o Agente 07 detecta um problema em produção, o `intent.md` original está acessível — o on-call entende o que o código deveria fazer antes de investigar por que está falhando.

### Alternativas Consideradas

**Artefatos apenas quando necessário (opcional):** Derrota o propósito. A disciplina dos artefatos é o que cria o rastro — torná-los opcionais significa não adotados. Times sob pressão pulam etapas.

**Documentação em Confluence/Notion:** Sem versionamento junto ao código, sem gatilho de CI quando desatualizada, e duplica a fonte da verdade (o código diz uma coisa, o Confluence diz outra).

**Tudo em comentários de PR:** PR comments não têm estrutura, não são pesquisáveis de forma eficiente, e ficam enterrados no histórico.

### Consequências

- O processo de desenvolvimento tem etapas formais que requerem ação deliberada antes de avançar — não é possível pular de card para código sem criar os artefatos intermediários
- O tempo entre "card aprovado" e "PR aberto" inclui a criação e aprovação de intent, spec e plan — o ganho de velocidade vem da qualidade maior, não de menos passos
- Incidentes de produção têm rastreabilidade completa — o on-call sabe o que o código deveria fazer

---

## ADR-004 — Evals para Cada Agente

### Contexto

Agentes de IA são configurados via prompt — `copilot-instructions.md` e skills. Qualquer mudança de configuração pode alterar o comportamento do agente de formas não óbvias. Um novo padrão adicionado ao `platform-standards` pode, inadvertidamente, fazer o Agente 03 (Plan) produzir planos com arquivos errados. Uma skill `bbds-api` atualizada pode fazer o Agente 02 (Spec) mapear componentes incorretamente.

Sem testes, mudanças de configuração são deploy no escuro.

**O que são evals?** Evals (abreviação de "evaluations") são testes para comportamento de agentes. Assim como testes unitários verificam que uma função retorna o resultado correto para inputs conhecidos, evals verificam que um agente produz o output correto para situações conhecidas. A diferença: testes unitários verificam código determinístico; evals verificam comportamento probabilístico de um modelo de linguagem — e por isso requerem graders mais sofisticados.

**pass@1 vs pass^k:**
- `pass@1`: o agente resolve a tarefa corretamente na primeira tentativa. Para tarefas rotineiras (gerar um intent.md a partir de um card simples), inconsistência é inaceitável — o agente precisa funcionar sempre, não "na maioria das vezes".
- `pass^k`: o agente resolve a tarefa corretamente na maioria de k tentativas. Para tarefas complexas (diagnóstico de incidente com correlação de múltiplas fontes), consistência em múltiplas tentativas é o que importa — não o sucesso ocasional.

### Decisão

Cada agente tem uma suite de 20–50 evals baseadas em casos reais. Mudanças em `copilot-instructions.md` ou skills disparam `eval-suite.yml` em CI antes do merge. Pass rate global ≥ 85% é o gate de avanço.

### Justificativa

**A analogia com testes de software é direta.** Skills e instruções são código de configuração de agentes. Código sem testes é dívida técnica. Configuração de agentes sem evals é o mesmo risco: mudanças silenciosas que quebram comportamento sem ninguém perceber.

**Graders em camadas reduzem custo sem perder cobertura:**

- **Code-based** (regex, schema check, binary pass/fail): rápidos, sem custo de modelo, determinísticos — usados para verificar estrutura, campos obrigatórios, schema de artefatos
- **Model-based** (rubrica 1-5 aplicada por um modelo): avalia qualidade semântica onde regex não chega — "o diagnóstico é fundamentado?" não tem resposta binária
- **Human** (revisão manual): usado para calibrar os model-based graders inicialmente; não escala para CI, mas é a origem do ground truth

**Incidentes de produção viram evals permanentes.** Quando um bug chega a produção que o agente deveria ter prevenido, o caso vira uma task de eval permanente na suite. "Não pode acontecer de novo" tem implementação concreta — não é apenas uma promessa de processo.

**O threshold é uma política de qualidade deliberada.** 85% de pass rate global significa que 15% das tarefas de eval podem falhar e o pipeline ainda avança. Esse número não é fixo — o tech lead define e aprova. Reduzir o threshold para "fazer a CI passar" seria equivalente a deletar testes: requer decisão consciente de quem tem autoridade técnica, não unilateral.

### Alternativas Consideradas

**Testes manuais antes de cada deploy de configuração:** Não escala. O time de plataforma não pode revisar manualmente cada mudança em todos os repos de dezenas de times.

**Sem evals — confiar no Copilot:** O comportamento do agente muda com mudanças de skills/instruções de formas não óbvias. Regressões só seriam detectadas quando um engenheiro ou revisor humano percebesse output incorreto — tarde demais.

**Evals apenas em releases maiores:** Delays na detecção de regressões. Mudanças incrementais acumuladas sem validação podem produzir comportamentos inesperados que são difíceis de rastrear depois.

### Consequências

- CI fica mais lento para mudanças em configuração de agentes (eval suite roda adicional)
- Cada nova funcionalidade adicionada a um agente requer tasks de eval correspondentes — custo de desenvolvimento mais alto, mas qualidade garantida
- A confiança no comportamento dos agentes é quantificada e acompanhada ao longo do tempo — o time sabe exatamente quão confiável cada agente está

---

## ADR-005 — Fase 0 como Pré-Requisito Bloqueante

### Contexto

Dezenas de repos existem sem `copilot-instructions.md`, sem padrões documentados, sem contexto de plataforma formal. O conhecimento de plataforma — arquitetura de bundle, anti-patterns, convenções — vive na cabeça de engenheiros sênior e em documentações Confluence desatualizadas.

A tentação é iniciar os agentes o quanto antes, adicionando contexto incrementalmente conforme o time percebe que falta.

### Decisão

A Fase 0 (Bootstrap de Conhecimento de Plataforma) é executada antes de qualquer agente entrar em produção. Sem ela, nenhum repo é incluído na esteira.

### Justificativa

**Agentes operaram com o contexto que recebem — e o contexto determina a qualidade do output.**

Um Agente 04 (Build) sem contexto de plataforma vai:
- Gerar código que viola convenções de nomenclatura do bundle
- Usar bibliotecas fora da allowlist aprovada
- Ignorar anti-patterns conhecidos que causaram problemas antes
- Criar estruturas de pastas incorretas

O resultado: o code review humano precisa capturar todos esses problemas — cancelando o ganho de velocidade que a esteira prometia. Pior: cria a percepção de que "o Copilot gera código ruim", o que é difícil de reverter organizacionalmente.

**A Fase 0 não é overhead — é o fundamento.**

O bootstrap é progressivo e priorizado: repos com maior volume de mudanças recebem atenção primeiro. O nível mínimo (um `copilot-instructions.md` base com estrutura, comandos e anti-patterns críticos) leva ~2 semanas por cluster de repos e pode ser feito em paralelo por múltiplos engenheiros de plataforma.

O investimento se paga nas primeiras semanas de uso dos agentes: menos ciclos de review, menos rework, menos problemas básicos passando pelo processo.

**O bootstrap valida o conhecimento de plataforma ao mesmo tempo.**

Formalizar os padrões em skills cria oportunidade de identificar inconsistências entre o que diferentes engenheiros consideram "padrão". O processo de review dos skills (via PR) é em si uma forma de alinhar o time sobre as convenções — independente do uso de agentes.

### Alternativas Consideradas

**Adicionar contexto incrementalmente (começar sem Fase 0):** O argumento é velocidade — iniciar os agentes imediatamente e adicionar skills conforme percebemos que faltam. O problema: os primeiros outputs dos agentes são ruins, criam resistência à adoção, e a reputação de "o agente não funciona bem aqui" é difícil de reverter mesmo depois que o contexto está completo.

**Context window grande como substituto de skills:** Jogar toda a documentação disponível no contexto de cada sessão não é escalável (contextos longos aumentam latência e custo), e é menos eficaz do que skills curadas — contexto irrelevante dilui o sinal útil.

### Consequências

- Há um período de 4-6 semanas de investimento em conhecimento antes do primeiro agente funcionar bem em produção
- O processo de bootstrap revela gaps de documentação e inconsistências de padrões — é uma oportunidade de alinhamento do time
- Repos sem Fase 0 completa ficam fora da esteira — cria incentivo para priorizar o bootstrap

---

## ADR-006 — BBDS como Infraestrutura Auto-gerada (Fase 0b)

### Contexto

O BBDS é o design system obrigatório para novos fluxos React Native. Ele tem releases frequentes — novas versões trazem novos componentes, novos props, e props que ficam deprecated com migration paths para os novos.

Sem conhecimento atualizado da API do BBDS, o Agente 04 (Build) usa props incorretas ou obsoletas. Isso cria um ciclo:

```
Agente 02 especifica componente BBDS com prop errada
→ Agente 04 implementa com a prop errada
→ Agente 06 (code review) não detecta (prop não é óbvia)
→ Bug visual ou runtime error em produção
→ Rework no ciclo seguinte
```

Documentação manual do BBDS não é sustentável: a cada release, alguém precisa ler o changelog, atualizar o documento, e fazer review. Na prática, isso não acontece — a documentação fica para trás.

### Decisão

`bbds-api-reference.md` é gerado automaticamente via `ts-morph` a partir dos TypeScript types do BBDS cada vez que há um version bump no `package.json`. A documentação de UX (`bbds-ux-guidelines.yaml`) é curada separadamente pelo time de UX. Três skills resultantes: `bbds-api`, `bbds-ux-guidelines`, `bbds-patterns`.

### Justificativa

**O que é ts-morph e por que funciona aqui:**

`ts-morph` é uma biblioteca Node.js que permite ler e analisar código TypeScript programaticamente — sem compilar o código, apenas parseando a AST (Abstract Syntax Tree). O BBDS já é escrito em TypeScript com tipos explícitos para cada prop (`ButtonProps`, `CardProps`, etc.) e JSDoc comments que documentam `@default`, `@deprecated`, e descrições.

O script `extract-bbds-types.ts` lê esses tipos e extrai:
- Quais props existem por componente
- O tipo de cada prop (`string`, `"primary" | "secondary"`, `boolean`)
- O valor default (de `defaultProps` ou da definição do tipo)
- Quais props têm `@deprecated` e qual é o migration path documentado no JSDoc

O resultado é um arquivo Markdown estruturado. O processo leva menos de 30 minutos e roda automaticamente em CI após cada bump de versão do BBDS — lag zero entre release e documentação atualizada.

**Separação de fontes por natureza do conhecimento:**

A API (props, tipos, defaults, deprecated) é conhecimento que **vive no código** — extraível programaticamente, sempre atualizado, sem intervenção humana.

Os guidelines de uso (quando usar `Button` vs `ActionButton`, quando não usar `Modal` em favor de um bottom sheet, combinações de componentes que criam experiências ruins) são conhecimento de **julgamento de UX** — não estão nos tipos TypeScript. Esse conhecimento é curado pelo time de UX e é mais estável (muda com decisões de design, não a cada release do BBDS).

Misturar as duas fontes num único mecanismo (auto-geração) resultaria em guidelines de UX que não existem nos types, ou num processo de curação manual que atrasa a atualização da API.

**O problema sem isso:**

Agente 02 especifica `<Button color="blue">` → ts-morph já extraiu que `color` está deprecated desde v3.1 → a skill `bbds-api` já tem o migration path → Agente 04 usa `<Button variant="primary">` automaticamente. Sem a skill auto-gerada, o agente usa o prop deprecated porque não há como saber que ele mudou.

### Alternativas Consideradas

**Documentação Storybook como contexto:** Storybook é ótimo para exploração visual, mas não tem toda a API documentada de forma estruturada para consumo por máquina. E fica desatualizado da mesma forma que documentação manual.

**Skill curada manualmente:** Exigiria atualização manual a cada release do BBDS — insustentável com a frequência de releases atual.

**Sem skill de BBDS:** Agentes usam conhecimento geral de React Native + tentativa e erro. Alta taxa de props incorretas e deprecated. Code review humano captura a maioria, mas com custo de ciclos adicionais.

### Consequências

- O processo de auto-geração precisa ser validado: o script `extract-bbds-types.ts` pode falhar se a estrutura dos types do BBDS mudar. Monitorar o workflow `bbds-api-sync.yml` em CI
- Props com documentação JSDoc incompleta no BBDS resultam em documentação incompleta na skill — melhoria da qualidade do JSDoc no BBDS melhora diretamente a qualidade dos agentes
- A skill `bbds-ux-guidelines.yaml` continua a ser curada pelo time de UX — requer processo de review e atualização quando padrões de UX mudam

---

## ADR-007 — Pipeline de Correlação para Monitoramento de Jornadas

### Contexto

A organização tem duas ferramentas de monitoramento de produção com sinais complementares:
- **Firebase Crashlytics**: detecta crashes técnicos com causa identificável (arquivo:linha no stacktrace)
- **Journey Monitor**: detecta quando usuários não completam fluxos negociais

O problema com falhas de jornada: o mesmo sinal ("taxa de conclusão do fluxo X caiu 34%") pode ter quatro causas completamente diferentes — e a ação correta é diferente para cada uma:

| Causa | Sinal adicional | Ação correta |
|---|---|---|
| Erro de endpoint | Journey Monitor detecta API com erro | Acionar time de backend, avaliar rollback de endpoint |
| Crash na tela problemática | Firebase tem crash na mesma tela/período | Corrigir bug técnico (stacktrace disponível) |
| Regressão por commit recente | git log tem commit nessa tela recentemente | Avaliar rollback do commit, acionar autor |
| Bug de lógica silencioso | Nenhum dos sinais acima | Investigação com session recording, hipótese aberta |

Se o Copilot recebe apenas o sinal de queda de jornada sem contexto adicional, ele precisa especular qual das quatro causas é a mais provável — e pode errar. Um diagnóstico incorreto direciona o time para a solução errada, desperdiçando tempo de on-call.

### Decisão

Falhas de jornada passam por um **pipeline de correlação determinístico** antes da análise do Copilot. A correlação cruza: erros de endpoint (fornecidos pelo Journey Monitor), crashes do Firebase na mesma tela/período, commits recentes tocando a tela problemática, e deploys recentes do bundle. O Copilot recebe o "correlation bundle" completo e segue uma árvore de diagnóstico com branches predefinidos.

### Justificativa

**Prevenção de hallucination no diagnóstico:**

Modelos de linguagem tendem a preencher gaps de informação com hipóteses plausíveis mas incorretas (hallucination). Sem correlação, o Copilot receberia "taxa de sucesso caiu 34% no fluxo X" e poderia especular: "provavelmente é problema de UX" ou "pode ser relacionado ao último deploy" — hipóteses não fundamentadas.

Com o correlation bundle, o Copilot recebe evidências concretas: "endpoint retornando 503 + commit abc123 em ConfirmacaoDadosScreen.tsx há 6 horas + deploy v2.4.1 há 8 horas". A hipótese é fundamentada em dados — não em especulação.

**A árvore de diagnóstico é testável em evals:**

Cada branch da árvore tem inputs e outputs claros. Evals podem validar cada caminho com casos históricos reais:
- Task "endpoint_error": signal com `endpoint_errors: [{code: 503}]` → grader verifica que o intent.md menciona o endpoint e direciona para o time correto
- Task "crash_found": signal sem endpoint error, com crash Firebase → grader verifica que o intent.md inclui o stacktrace e o arquivo:linha corretos
- E assim por diante para cada branch

Sem a árvore predefinida, o diagnóstico seria livre — impossível criar ground truth para evals.

**A granularidade de `screen_id` é o diferencial:**

O Journey Monitor já entrega qual tela específica do funil tem queda de navegação — não apenas "o fluxo X tem problema". Com o `screen_id` em mãos, a correlação com git log (`-- *ConfirmacaoDadosScreen*`) e Firebase (crashes naquela tela específica) é precisa. Sem essa granularidade, a correlação seria muito ampla para ser útil.

### Alternativas Consideradas

**Análise direta pelo Copilot sem correlação:** Alto risco de hipóteses incorretas. Evals seriam impossíveis de criar com ground truth. O on-call receberia diagnósticos que podem estar certos ou errados sem como distinguir.

**Cada fonte de monitoramento gera seu próprio intent.md independente:** O mesmo incidente poderia gerar 3 intents.md simultâneos (Firebase + Journey Monitor + CI). Cria ruído para o on-call que precisa triagear múltiplos items para o mesmo problema. A correlação une os sinais em uma hipótese única e coesa.

**Dashboard manual para o on-call correlacionar:** Remove o benefício de automação. O on-call ainda faz o trabalho intelectual de correlacionar Firebase + git + deploys manualmente a cada incidente — o que a esteira automatiza.

### Consequências

- A qualidade do diagnóstico depende da qualidade da correlação: se o Journey Monitor não consegue identificar o `screen_id` para um fluxo específico, a correlação com git log fica menos precisa
- O pipeline de correlação adiciona latência ao diagnóstico (3 passos paralelos em GitHub Actions) — o intent.md aparece em minutos, não segundos
- Casos que a árvore de diagnóstico não cobre (combinações de múltiplas causas simultâneas) caem no branch "silent failure" — investigação manual ainda necessária nesses casos

---

## ADR-008 — Gates de Governança Humana por Estágio

### Contexto

Agentes de IA cometem erros. A questão não é se — é com qual frequência e com qual impacto. O risco de um erro não é uniforme em todos os estágios do desenvolvimento:

| Estágio | Tipo de erro possível | Impacto se não detectado |
|---|---|---|
| intent.md | Problema mal definido, outcome incorreto | Equipe implementa a solução errada — ciclo desperdiçado |
| spec.md | Requisito faltando, flag de compliance ignorada | Bug de produto ou violação de compliance descoberta tarde |
| plan.md | Estratégia inviável, arquivo errado listado | Implementação que descobre no meio que não funciona |
| código | Bug, prop BBDS errada, sem tratamento de erro | Problema capturado em CI ou code review — custo médio |
| PR | Bug sutil não detectado pelo CI | Problema em produção — custo alto |
| deploy prod | Deploy no momento errado, sem rollback | Incidente com usuários — custo muito alto |

### Decisão

Cada estágio tem um gate humano explícito antes de avançar. Nenhum agente aprova seu próprio output. O nível de autonomia escala inversamente ao risco: dev (livre, CI automático), staging (automático com CI + 1 humano), prod (release manager nomeado).

### Justificativa

**Separação de funções — o mesmo princípio de auditoria de sistemas financeiros.**

O mesmo desenvolvedor que escreve o código não pode ser o único revisor. O mesmo agente que gera o `intent.md` não pode aprovar o `intent.md`. Essa separação não é burocracia — é o mecanismo que detecta erros de forma independente.

**Autonomia é conquistada progressivamente.**

A esteira começa conservadora: gates em todos os estágios, aprovação humana explícita. À medida que o pass rate dos evals sobe e a confiança no comportamento dos agentes cresce, alguns gates podem ser relaxados — por exemplo, auto-merge de intent.md de baixo risco após N dias sem contestação.

O caminho inverso — começar autônomo e adicionar gates após um incidente — é muito mais difícil. Incidentes criam resistência organizacional que pode matar a adoção da esteira inteira. "O Copilot deployou código errado em produção" é uma história que precisa ser evitada.

**Gates mapeiam para processos existentes — não criam burocracia nova.**

- PO aprova intent e spec: já faz isso hoje, informalmente
- Engenheiro commita o plan: já valida a abordagem hoje, mentalmente
- Release manager aprova deploy em prod: processo existente
- Copilot Code Review + 1 humano: review de PR já existe; o Copilot aumenta a cobertura

A esteira formaliza e torna explícito o que já acontecia implicitamente. O humano continua fazendo o mesmo julgamento — agora com mais contexto (o artefato do agente como ponto de partida) e de forma registrável.

**O gate de produção é permanente.**

Deploy para produção requer aprovação explícita do release manager via GitHub environment protection — não é um checkbox no PR. Isso permanece independente de quanto os agentes melhoram. O risco de um deploy em produção nunca é zero, e uma aprovação humana consciente é o último mecanismo de defesa.

### Alternativas Consideradas

**Full autonomy desde o início:** Velocidade máxima, mas um erro em produção cria resistência organizacional que pode matar a adoção da esteira inteira. O risco de curto prazo não justifica a velocidade.

**Gate único no final (antes do deploy):** Não detecta problemas cedo o suficiente. Um spec incorreto descoberto na revisão do PR representa rework de todo o ciclo de Build + Test + Deploy — o artefato errado se propagou por estágios inteiros antes de ser detectado.

**Gates apenas em produção:** Melhoria em relação ao anterior, mas perde o valor dos artefatos (intent, spec, plan) como pontos de decisão conscientes que criam o rastro de auditoria.

### Consequências

- O pipeline tem latência em cada gate — o tempo de aprovação humana é variável e não controlável pela esteira
- Gates explícitos criam registro de quem aprovou o quê e quando — trilha de auditoria completa para incidentes
- A autonomia pode aumentar gradualmente conforme confiança é construída — o design permite isso sem mudar a arquitetura

---

## ADR-009 — Arquitetura de Distribuição de Conhecimento (DECISÃO EM ABERTO)

### Contexto

A esteira precisa distribuir conhecimento de plataforma — padrões de código, API do BBDS, guidelines de UX, regras de segurança — para agentes que rodam em muitos repositórios, cada um mantido por equipes de produto autônomas.

O plano original (Fase 0) previa Skills (SKILL.md) como mecanismo de distribuição. Essa abordagem é adequada para qualidade e determinismo do contexto (ver [ADR-002](#adr-002--skills-ao-invés-de-rag-para-conhecimento-de-domínio)), mas apresenta um problema estrutural de distribuição em escala:

- Skills são arquivos em repos; cada time decide quando instalar e atualizar
- Sem enforcement centralizado, repos acumulam versões diferentes do mesmo padrão (configuration drift)
- A [GitHub Discussion #179641](https://github.com/orgs/community/discussions/179641) confirma que o Copilot Enterprise não tem solução nativa para este problema em multi-repo

A análise completa das alternativas está em [`docs/knowledge-governance.md`](knowledge-governance.md).

### Alternativas em análise

**A — Org-level Copilot Instructions** (disponível agora, zero infraestrutura):
Regras não-negociáveis configuradas pelo admin da org, aplicadas a todos os repos automaticamente. Resolve o problema para conteúdo limitado; não suporta skills ricas como a API completa do BBDS.

**B — MCP Server Centralizado** (modelo Stripe):
Servidor que expõe conhecimento via endpoints determinísticos (`get_component_api`, `get_standard`). Agentes consultam o servidor em tempo de sessão — nenhuma cópia local em repos. Zero drift. Auditabilidade total via logs. Requer nova infraestrutura.

**C — Pacote npm Versionado** (modelo Shopify):
Conhecimento empacotado como dependência. Times instalam via `npm install @platform/knowledge-skills`. Dependabot automatiza PRs de update. Drift possível (time pode ignorar Dependabot), mas explícito e rastreável.

**D — ContextOps com Push Automático**:
GitHub Actions abre PRs em todos os repos a cada atualização de padrão. Dashboard de drift detection mostra quais repos estão desatualizados. Times ainda fazem merge, mas o processo é automatizado até a porta.

### Recomendação preliminar

Para muitas equipes descentralizadas em contexto bancário, a arquitetura híbrida em três camadas é a mais adequada:

1. **Org-level instructions** — regras críticas, enforcement automático, zero infraestrutura
2. **MCP server central** — conhecimento rico (BBDS, platform standards), zero drift, auditabilidade
3. **Per-repo copilot-instructions.md** — contexto bundle-específico que o time genuinamente controla

### Decisão

**Não definida.** A escolha da arquitetura requer validação de:
- Viabilidade de construir e operar o MCP server internamente
- Aceitação dos times de produto do modelo de consulta centralizada
- Avaliação de MCP Gateway para auditoria regulatória
- Estratégia de faseamento (org-level instructions no MVP; MCP server em seguida?)

**Participantes necessários:** time de plataforma, arquitetura, segurança.

---

## ADR-010 — Agente 01 (Intent) fora do Escopo da Esteira Mobile

### Contexto

O ciclo completo de desenvolvimento começa com a identificação e definição formal do problema — o `intent.md`. Em um ciclo AI-native, esse artefato pode ser gerado por um agente que lê o card de backlog (BusinessMap) e o transforma em documento estruturado.

A plataforma mobile opera dentro de uma organização maior que tem uma área negocial e uma plataforma de agilidade dedicadas à gestão de backlog e refinamento de problemas. Essas áreas já têm processo para aprovação de demandas antes de chegarem ao desenvolvimento.

A questão colocada: **a criação do `intent.md` deve ser parte da esteira da plataforma mobile, ou é responsabilidade da área negocial/agilidade?**

### Decisão

A criação e aprovação do `intent.md` é **responsabilidade da área negocial em conjunto com a plataforma de agilidade** — fora do escopo de implementação da esteira mobile.

A esteira mobile começa no **Agente 02 (Spec)**, recebendo o `intent.md` como input já aprovado. A plataforma mobile define o **schema** (contrato de interface) que o `intent.md` deve seguir para ser aceito pela esteira.

### Justificativa

**Separação de responsabilidades organizacional:**

A decisão "qual problema resolver e por quê" pertence à área negocial. Essa decisão envolve priorização estratégica, alinhamento com objetivos de negócio e contexto que vai além do scope técnico da plataforma mobile. Trazer essa decisão para dentro da esteira mobile criaria uma sobreposição de responsabilidades — a plataforma mobile seria responsável por gerar um documento de negócio que outras áreas são as legítimas donas.

**O processo já existe:**

A área negocial e a plataforma de agilidade já têm fluxo de refinamento e aprovação de demandas. Criar um agente que duplica esse processo dentro da esteira mobile seria redundante e poderia criar conflito sobre qual `intent.md` é o autoritativo.

**O schema como contrato suficiente:**

O que a esteira mobile precisa não é controlar como o `intent.md` é criado — precisa garantir que o `intent.md` que chega tenha os campos necessários para o Agente 02 gerar uma spec de qualidade. Isso é resolvido pelo schema (contrato de interface) e pela validação automática no `intent-to-spec.yml`.

**Flexibilidade para a área negocial:**

A área negocial pode usar um agente (como o Agente 01 documentado em `docs/agents/01-intent.md`) para criar o `intent.md` a partir do BusinessMap MCP, ou pode criá-lo manualmente, ou via outra ferramenta — desde que o schema seja seguido. A decisão sobre como criar o `intent.md` é da área negocial.

### Alternativas Consideradas

**Incluir Agente 01 no escopo da plataforma mobile:** A plataforma mobile teria controle sobre o início do pipeline. Mas criaria sobreposição com o processo da área negocial/agilidade, e a plataforma mobile precisaria manter integração com BusinessMap que hoje pertence a outra área.

**Nenhum schema — aceitar qualquer intent.md:** Flexibilidade máxima para a área negocial, mas o Agente 02 receberia inputs inconsistentes e de qualidade variável, gerando specs incompletas ou incorretas.

### Consequências

- A esteira mobile tem um **pré-requisito externo** claro: um `intent.md` aprovado com schema válido
- O schema (`templates/intent.md`) é o **artefato de integração** entre a área negocial e a plataforma mobile — precisa ser acordado e mantido conjuntamente
- O workflow `intent-to-spec.yml` valida o schema automaticamente, dando feedback imediato quando um `intent.md` não atende o contrato
- A documentação completa do schema e da integração com BusinessMap MCP está em `docs/agents/01-intent.md` — disponível para a área negocial/agilidade usar como referência ao implementar seu próprio processo de geração de intent
- Incidentes detectados pelo Agente 07 geram `intent.md` automaticamente — esses passam pela triagem do on-call antes de entrar na fila da área negocial/agilidade para priorização

**Bloqueador:** Fase 0 não entra em produção sem essa decisão — o mecanismo de distribuição precede a criação do conteúdo.
