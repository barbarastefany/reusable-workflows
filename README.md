# Reusable Workflows

Este repositório centraliza workflows reutilizáveis do GitHub Actions para padronizar rotinas de build, testes e criação de Pull Requests entre branches (feature → develop, develop → main).

## Objetivo
- Padronizar o pipeline de build e testes Java/Maven.
- Automatizar a abertura de PRs entre branches padrão do repositório.
- Facilitar a reutilização em múltiplos repositórios sem duplicar lógica.

## Pré-requisitos
1. GitHub Actions habilitado no repositório consumidor:
   - Vá em Settings → Actions → General.
   - Em "Actions permissions", selecione "Allow all actions and reusable workflows".
   - Em "Workflow permissions", selecione "Read repository contents permission" e marque "Allow GitHub Actions to create and approve pull requests" quando for utilizar os workflows de PR (isso concede `pull-requests: write`).
2. O repositório consumidor precisa ter as branches envolvidas (ex.: `develop`, `main`), e permissões para o `GITHUB_TOKEN` criar PRs.
3. Caso precise permissões extras (cross-repo ou orgs restritas), use um Personal Access Token (PAT) com escopos adequados e passe via input `github-token`.

## Workflows disponíveis

### 1. Build, Testes e abertura de PR para develop (Maven)
Arquivo: `.github/workflows/1-build-test.yml`

- Inputs:
  - `jdk-version` (padrão: 17)
  - `maven-args` (argumentos opcionais adicionais, ex.: `-DskipTests`)
  - `source-branch` (padrão: `feature/sua-branch`)
  - `target-branch` (padrão: `develop`)

Exemplo de uso em um repositório consumidor:
```yaml
name: Build, Test and Create PR to develop

on:
  push:
    branches:
      - "feature/**"
  workflow_dispatch:

permissions:
  contents: read
  pull-requests: write

jobs:
  build_and_test:
    uses: barbarastefany/reusable-workflows/.github/workflows/1-build-test.yml@main
    with:
      jdk-version: "17"
      maven-args: ""

  create_pr_to_develop:
    needs: build_and_test
    uses: barbarastefany/reusable-workflows/.github/workflows/2-pr-to-develop.yml@main
    with:
      source-branch: "${{ github.ref_name }}"
      target-branch: "develop"
    secrets: inherit
```

### 2. PR para Main
Arquivo: `.github/workflows/3-pr-to-main.yml`

- Inputs:
  - `source-branch` (padrão: `develop`)
  - `target-branch` (padrão: `main`)

Exemplo que cria PR de develop para main automaticamente quando um PR para develop é mergeado:
```yaml
name: Create PR develop to main on merge

on:
  pull_request:
    branches: [develop]
    types: [closed]

jobs:
  create_pr_develop_to_main:
    if: ${{ github.event.pull_request.merged == true }}
    permissions:
      contents: write
      pull-requests: write
    uses: barbarastefany/reusable-workflows/.github/workflows/3-pr-to-main.yml@main
    with:
      source-branch: "develop"
      target-branch: "main"
    secrets: inherit
```

## Permissões e segurança
- Os workflows de PR requerem `pull-requests: write`. Configure em Settings → Actions → General → Workflow permissions.
- O workflow de build usa apenas `contents: read` (padrão) para checkout.
- Para cenários com forks ou cross-repo, considere usar um PAT com escopos `repo` e passá-lo via `secrets` como input `github-token`.

## Dicas e solução de problemas
- "Branch alvo/origem não encontrada": garanta que as branches existem no repositório consumidor.
- "PR já existe": o workflow detecta PRs abertos para evitar duplicidade.
- Erros de permissão: verifique as permissões do `GITHUB_TOKEN` e se a organização permite reusables (`Allow all actions and reusable workflows`).
- Maven em multi-módulos: use `working-directory` para apontar para o submódulo correto.

## Como versionar e consumir
- Recomenda-se fixar a referência com uma tag ou SHA:
  - `uses: barbarastefany/reusable-workflows/.github/workflows/1-build-test.yml@v1`
- Atualize o consumidor quando novas versões/ajustes forem publicados.