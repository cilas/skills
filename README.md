# Skills, MCPs e Git Worktrees: Um Fluxo de Implementação com Múltiplos Agentes

> Como combinar skills de Claude Code, servidores MCP e git worktrees para criar um pipeline de desenvolvimento paralelo, rastreável e livre de conflitos.

---

## Introdução

A adoção de agentes de IA no desenvolvimento de software levanta uma questão prática: como fazer vários agentes trabalharem no mesmo repositório sem se atrapalharem? Branches isolam código, mas não isolam o sistema de arquivos. Um agente modificando um arquivo enquanto outro lê o mesmo arquivo é receita para conflito.

Este artigo apresenta uma solução que combina três conceitos:

1. **Skills** — instruções reutilizáveis que ensinam o agente a se comportar
2. **MCPs** (Model Context Protocol) — integrações que dão ao agente acesso a ferramentas externas (Linear, GitHub)
3. **Git Worktrees** — espelhos do repositório em diretórios separados, cada um em seu próprio branch

O resultado é um fluxo onde cada tarefa vive em seu próprio diretório isolado, rastreada no Linear, com PR aberta automaticamente via GitHub — tudo orquestrado por uma única skill chamada `git-linear-flow`.

---

## O que é uma Skill no Claude Code

Uma **skill** é um arquivo Markdown com frontmatter YAML que o Claude Code carrega como instrução de comportamento. Funciona como um "modo" ou "papel" que o agente assume ao executar uma tarefa.

### Estrutura de uma skill

```markdown
---
name: git-linear-flow
description: Linear + Git integration workflow. MANDATORY for any implementation task.
---

# Git Flow

## Regras
1. NUNCA commitar direto na `main`.
2. Toda tarefa = uma nova worktree a partir da `main` atualizada.
...
```

O frontmatter define metadados (nome, descrição). O corpo Markdown define **o que o agente deve fazer**, em que ordem, e quais ferramentas usar.

### Onde ficam as skills

Skills podem ser armazenadas em três lugares:

| Local | Escopo |
|-------|--------|
| `~/.claude/skills/` | Globais — disponíveis em qualquer projeto |
| `repo/.claude/skills/` | Por repositório — versionadas com o código |
| Plugins instalados | Distribuídas via marketplace |

Skills de repositório são especialmente poderosas: elas viajam com o código, garantindo que qualquer desenvolvedor (ou agente) que clone o repo opere da mesma forma.

### Como invocar uma skill

No Claude Code, skills são invocadas com o prefixo `/`:

```
/git-linear-flow CIL-42 — Implementar autenticação por JWT
```

O agente lê a skill, carrega as instruções e executa o fluxo definido — incluindo chamadas a MCPs, criação de branches, abertura de PRs, etc.

---

## O que é um MCP (Model Context Protocol)

**MCP** é o protocolo que permite ao Claude Code se conectar a ferramentas externas de forma padronizada. Em vez de o agente apenas "sugerir" comandos, ele os executa diretamente através de servidores MCP.

Exemplos de servidores MCP:

- **Linear MCP** — lê e atualiza issues, muda status, adiciona comentários
- **GitHub MCP** — cria branches, abre PRs, lista repositórios
- **Filesystem MCP** — lê e escreve arquivos com permissões controladas

### Por que MCPs importam para o fluxo de features

Sem MCP, o agente precisaria pedir ao desenvolvedor para "abrir uma PR" ou "atualizar o status no Linear". Com MCP, o agente faz isso diretamente como parte do fluxo — sem interrupção humana.

Isso transforma o agente de um **assistente de código** em um **membro do time** que segue o processo de desenvolvimento de ponta a ponta.

---

## O que é um Git Worktree

Um **git worktree** é uma cópia de trabalho adicional de um repositório. O repositório central permanece intacto, mas você pode ter múltiplos diretórios, cada um em um branch diferente, todos compartilhando o mesmo histórico git.

```bash
# Repositório principal
~/code/meu-projeto/          # branch: main

# Worktrees adicionais
~/code/meu-projeto-feat-A/   # branch: feature/CIL-42-auth
~/code/meu-projeto-feat-B/   # branch: feature/CIL-55-dashboard
```

### Por que worktrees e não branches comuns?

Com branches comuns, trocar de branch (`git checkout`) muda os arquivos no mesmo diretório. Isso significa:

- Um agente trabalhando em `feature/A` pode ser interrompido se você trocar para `feature/B`
- Dois agentes no mesmo diretório entram em conflito imediatamente
- Ferramentas que observam o sistema de arquivos (compiladores, watchers) ficam confusas

Com worktrees, cada feature tem seu próprio diretório. Dois agentes podem trabalhar em paralelo sem interferência alguma.

### Comandos essenciais

```bash
# Criar worktree nova com branch nova a partir de main
git worktree add ../meu-projeto-CIL-42 -b feature/CIL-42-auth main

# Listar worktrees ativas
git worktree list

# Remover worktree após merge
git worktree remove ../meu-projeto-CIL-42
git worktree prune
git branch -d feature/CIL-42-auth
```

---

## A Skill `git-linear-flow` na Prática

A skill `git-linear-flow` usada neste projeto foi construída para ser o "cérebro do processo" de implementação. Ela orquestra Linear MCP, GitHub MCP e git worktrees em um fluxo de 6 fases.

### Arquivo da skill: `.github/skills/git-linear-flow/SKILL.md`

```markdown
---
name: git-linear-flow
description: Linear + Git integration workflow. MANDATORY for any implementation task.
---

# Git Flow — Linear + Git Workflow

## Regras

1. **NUNCA** realizar alterações direto na `main`.
2. Toda tarefa = uma nova worktree a partir da `main` atualizada.
3. Link do Linear fornecido → **BLOQUEAR** até ler via MCP. Se MCP falhar → PARAR.
4. PR exige: `lint` + `tsc --noEmit` + `test:unit` passando.
5. Commits atômicos (uma mudança lógica cada).
6. Manter o Linear atualizado (status + comentários).
```

### Fase 1 — Ler a Issue no Linear

A primeira coisa que o agente faz é usar o **Linear MCP** para ler a issue referenciada:

```
Linear MCP → get_issue(CIL-42)
→ Título, descrição, critérios de aceitação, prioridade
→ Atualiza status para "In Progress"
→ Adiciona comentário: "Iniciando implementação"
```

Se o MCP não consegue acessar a issue, **o agente para**. Não há implementação sem contexto.

Isso garante que o agente nunca implemente com base em um entendimento incompleto ou desatualizado do requisito.

### Fase 2 — Criar a Worktree

```bash
# Garantir que main está atualizada
git checkout main && git pull origin main

# Criar worktree isolada
git worktree add ../projeto-CIL-42-auth -b feature/CIL-42-auth main

# Entrar na worktree
cd ../projeto-CIL-42-auth

# Instalar dependências (isoladas neste diretório)
bun install
```

Neste ponto, o agente está em um diretório completamente isolado. O repositório principal não é afetado. Outros agentes podem criar suas próprias worktrees em paralelo.

### Fase 3 — Implementar

O agente implementa a feature seguindo a arquitetura do projeto. Commits são atômicos e seguem Conventional Commits:

```
feat(auth): add JWT token generation in LoginUseCase

Implements JWT signing using jose library with 7-day expiration.
Token payload includes userId and role for downstream authorization.

Ref: CIL-42
```

A referência `Ref: CIL-42` rastreia cada commit na issue do Linear.

### Fase 4 — Validação Antes do PR

Antes de abrir qualquer PR, a skill exige que três checks passem:

```bash
bun run lint        # Zero erros de lint
bunx tsc --noEmit   # Zero erros de TypeScript
bun test:unit       # Todos os testes novos passando
```

Se qualquer check falhar, o agente corrige antes de continuar. PRs com código quebrado não são abertas.

### Fase 5 — Abrir o PR via GitHub MCP

```bash
git push -u origin feature/CIL-42-auth
```

Em seguida, o **GitHub MCP** cria o PR automaticamente, usando o template `.github/PULL_REQUEST_TEMPLATE.md`:

```markdown
## Motivação
Implementar autenticação JWT para proteger rotas do dashboard.

## O que mudou
- LoginUseCase agora gera token JWT com `jose`
- Middleware valida token em todas as rotas protegidas
- Cookie HTTP-Only com flag Secure + SameSite=Lax

## Resolve: [CIL-42](https://linear.app/workspace/issue/CIL-42)
```

### Fase 6 — Atualizar Linear e Limpar Worktree

Após a PR ser mergeada:

```bash
# De volta ao repositório principal
git worktree remove ../projeto-CIL-42-auth
git worktree prune
git branch -d feature/CIL-42-auth
```

E no Linear:
- Status atualizado para "Done"
- Comentário final com link da PR e decisões tomadas

---

## Trabalhando com Múltiplos Agentes em Paralelo

O poder real da combinação skill + worktree aparece quando você roda múltiplos agentes simultaneamente.

### Exemplo: 3 features em paralelo

```
Agente 1 → /git-linear-flow CIL-42 (autenticação)
  └── worktree: ../projeto-CIL-42-auth/
      branch: feature/CIL-42-auth

Agente 2 → /git-linear-flow CIL-55 (dashboard financeiro)
  └── worktree: ../projeto-CIL-55-dashboard/
      branch: feature/CIL-55-dashboard

Agente 3 → /git-linear-flow CIL-61 (relatórios)
  └── worktree: ../projeto-CIL-61-reports/
      branch: feature/CIL-61-reports
```

Cada agente:
- Lê sua própria issue no Linear
- Trabalha em seu próprio diretório
- Commita em seu próprio branch
- Abre seu próprio PR

Sem conflitos. Sem interferência. O repositório principal (`main`) permanece limpo até que cada PR seja revisada e mergeada.

### Por que isso funciona melhor que branches simples

| Cenário | Branch comum | Worktree |
|---------|-------------|---------|
| Dois agentes simultâneos | ❌ Conflito de arquivos | ✅ Diretórios isolados |
| Trocar de contexto | ❌ `git stash` obrigatório | ✅ Só mudar de diretório |
| Build/watch em paralelo | ❌ Mesmo `node_modules` | ✅ Dependências isoladas |
| Rollback fácil | ❌ Cuidado com estado | ✅ `git worktree remove` |

---

## Anatomia Completa da Skill `git-linear-flow`

Para referência, a skill completa que orquestra tudo isso:

```markdown
---
name: git-linear-flow
description: Linear + Git integration workflow. MANDATORY for any implementation task.
---

## Nomenclatura de Branches

Padrão: `<prefixo>/CIL-XX-slug` → worktree: `../projeto-CIL-XX-slug`

| Prefixo    | Quando usar              |
|------------|--------------------------|
| `feature/` | Nova funcionalidade      |
| `bugfix/`  | Correção de bug          |
| `hotfix/`  | Correção urgente em prod |
| `refactor/`| Sem mudança de comportamento |
| `chore/`   | Ferramentas, deps, config |

## Fluxo Resumido

1. Ler issue no Linear via MCP (OBRIGATÓRIO)
2. `git checkout main && git pull`
3. `git worktree add ../projeto-CIL-XX -b feature/CIL-XX main`
4. Implementar com commits atômicos
5. `lint + tsc + test` — todos devem passar
6. `git push` + PR via GitHub MCP
7. Limpar worktree após merge
```

---

## Benefícios desse Fluxo

### Para o desenvolvedor

- **Rastreabilidade completa**: cada commit referencia uma issue; cada PR tem contexto
- **Menos overhead manual**: o agente cuida do Linear, do GitHub e do git
- **Parallelismo real**: múltiplos agentes, múltiplas features, zero conflitos

### Para o time

- **Processo documentado como código**: a skill está versionada no repositório
- **Padrão enforçado**: o agente não pula etapas — lint, tipos e testes são obrigatórios
- **Histórico auditável**: Linear registra decisões; GitHub registra código

### Para os agentes

- **Contexto garantido**: nunca implementa sem ler o requisito
- **Isolamento de sistema de arquivos**: worktrees eliminam interferência entre tarefas paralelas
- **Feedback loop fechado**: valida antes de abrir PR, comenta no Linear depois

---

## Conclusão

Skills, MCPs e git worktrees são três primitivos independentes. Cada um resolve um problema diferente:

- **Skills** resolvem o problema de comportamento: como o agente deve agir?
- **MCPs** resolvem o problema de integração: o agente tem acesso a ferramentas reais
- **Worktrees** resolvem o problema de isolamento: múltiplos agentes, sem conflitos

Combinados em uma skill como `git-linear-flow`, eles formam um pipeline de desenvolvimento que é ao mesmo tempo automatizado, rastreável e escalável — pronto para rodar quantos agentes em paralelo o projeto precisar.

O código resultante não é apenas funcional. Ele chega com contexto, com histórico e com todos os checks obrigatórios já validados.
