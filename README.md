# Engineering guardrails

Template (copier) e padrões de guardrails automatizados de qualidade de código para **Python, React/TypeScript, SQL e Terraform**, aplicados com **prek**, **GitHub Actions** e **GitLab CI**, para manter estruturado o código gerado por agentes de IA.

## Por que isso existe

Agentes de IA amplificam o ambiente em que trabalham: em um codebase sem regras objetivas, eles replicam a bagunça em velocidade industrial. Este repositório concentra as regras que o agente **não consegue contornar**: linters, tipos, testes de arquitetura, limites de tamanho e complexidade. O agente roda as verificações, vê o erro e se corrige sozinho. Esse ciclo de feedback é o que mais melhora a qualidade na prática.

## Princípios

- **Um comando único.** Tudo roda com `prek run --all-files`. O CI, os hooks locais e os hooks dos agentes chamam exatamente o mesmo conjunto de verificações.
- **Autocontido.** O template é copiado para dentro de cada repositório. Nada depende de acesso a este repositório em tempo de execução, o que permite usar em repositórios de clientes.
- **Portável entre plataformas.** Os pipelines do GitHub Actions e do GitLab CI são wrappers finos que apenas chamam o comando único.
- **Funciona em qualquer sistema operacional e em runners restritos.** Sem `make`, e com preferência por hooks `local` e `builtin`.
- **Bloqueia em qualquer violação.** Sem avisos: ou passa, ou falha.
- **Somente ferramentas open source gratuitas.**

## O que está incluído

| Camada | Verificações |
|---|---|
| **Python 3.13** | Ruff (lint e format, incluindo regras de complexidade e tamanho), mypy em modo strict, pytest, cobertura mínima em código novo, import-linter (arquitetura em camadas) |
| **React / TypeScript** (Node 24, React 19, Vite) | ESLint (typescript-eslint strict, react-hooks, jsx-a11y), Prettier, `tsc --noEmit`, Vitest, cobertura mínima em código novo, checagem de desvio do cliente gerado a partir do OpenAPI do FastAPI |
| **SQL** | SQLFluff configurado por dialeto em cada projeto, mais testes do dbt quando houver |
| **Terraform 1.16** | `terraform fmt`, `terraform validate`, TFLint, Trivy (scan de IaC), `required_version` e lockfile versionados. `terraform plan` apenas quando o cliente libera credenciais |
| **Transversal** | Gitleaks (segredos), Semgrep CE (SAST), script de tamanho de arquivo e de PR |

Gerenciador de dependências Python: **uv**. Aplicações Streamlit e NiceGUI **não são permitidas**.

## Limites objetivos

| Limite | Valor | Como é verificado |
|---|---|---|
| Complexidade ciclomática | ≤ 10 | Ruff `C901` e ESLint `complexity` |
| Tamanho de função | ≤ 50 statements (Python) e ≤ 80 linhas (TypeScript) | Ruff `PLR0915` e ESLint `max-lines-per-function` |
| Tamanho de arquivo | ≤ 500 linhas | Script local (Python) e ESLint `max-lines`. Exclui testes e código gerado |
| Tamanho de PR | ≤ 400 linhas alteradas | Script local. Exclui lockfiles, arquivos gerados e migrations |
| Cobertura em código novo | ≥ 80% (linhas e branches) | `diff-cover` |

## Arquitetura padrão dos backends

Backends FastAPI seguem **camadas pragmáticas**: `router → service → repository`, com os modelos de domínio isolados. As regras (por exemplo, "router não importa repository" e "o domínio não importa FastAPI nem banco") são contratos do `import-linter` e falham no CI quando violadas. Pipelines ETL seguem a estrutura extract, transform, load, com transformações puras e sem I/O.

## Como usar em um projeto novo

Pré-requisito: [uv](https://docs.astral.sh/uv/) instalado.

```bash
# 1. Gerar o projeto a partir do template (escolha os módulos: python, react, sql, terraform)
uvx copier copy <url-do-repositorio> ./meu-projeto

# 2. Instalar os hooks locais
cd meu-projeto
uvx prek install

# 3. Rodar todas as verificações
uvx prek run --all-files
```

O template é modular: as perguntas feitas na geração definem quais camadas entram no projeto, e ele funciona em repositórios com mais de uma stack.

## Atualizando um projeto existente

Quando o padrão evoluir, traga as mudanças como um diff revisável:

```bash
uvx copier update
```

Como o template é copiado e não referenciado, os repositórios podem divergir com o tempo. Atualize periodicamente.

## Como funciona sem branch protection

Em repositórios de clientes, nem sempre é possível configurar branch protection ou required checks. Nesse caso, as proteções são em camadas:

1. **Hooks locais** (pre-commit e pre-push) rodam o `prek` antes do código sair da máquina.
2. **Hooks dos agentes** (Claude Code e Kiro) rodam o `prek` após cada edição e ao concluir uma tarefa, para o agente se corrigir antes de chegar ao CI.
3. **CI vermelho** é o sinal que o time deve respeitar. Sem branch protection ele não impede tecnicamente o merge, então a regra é cultural: PR com CI vermelho não é mergeado.

## Exceções

Supressões são permitidas **somente inline e com justificativa obrigatória** (por exemplo `# noqa: C901 -- motivo`). Supressões genéricas, sem código de regra ou sem motivo, falham no CI.

## Estrutura do repositório

```text
engineering-guardrails/
├── README.md
├── docs/                 # Documento de padrões (o quê e por quê)
└── template/             # Template do copier, modular por stack
    ├── copier.yml
    └── ...               # prek, workflows do GitHub e GitLab, pyproject, ESLint, TFLint, hooks de agentes
```

## Fora do escopo por enquanto

- Tratamento do banco de vulnerabilidades (dependências vulneráveis e licenças).
- Mapeamento completo das categorias do OWASP.

Ambos estão adiados e serão tratados em uma próxima etapa.

## Contribuindo

Mudanças nos padrões passam por pull request, com justificativa registrada em `docs/`. Toda regra nova deve ter um motivo explícito e deve ser verificável automaticamente.
