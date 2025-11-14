# Estratégia de Centralização de CI/CD com GitHub Actions

Este documento descreve uma arquitetura para gerenciar pipelines de CI/CD em larga escala (300+ repositórios) usando **Workflows Reutilizáveis** (Reusable Workflows) e **Ações Compostas** (Composite Actions) do GitHub.

O objetivo é centralizar a lógica de pipeline em um único repositório, permitindo que as aplicações "consumam" esses workflows de forma versionada. Isso resolve o problema de ter que atualizar centenas de repositórios ao trocar uma ferramenta de scan (ex: Sonar por Veracode).

## 🚀 A Estratégia Principal

A arquitetura se baseia em dois tipos de repositórios:

1. **O Repositório "Centro de Controle" (ex: `shared-workflows`):**
   - Este repositório contém toda a lógica de pipeline.
   - Ele expõe **Workflows Reutilizáveis** (os "Orquestradores") que definem os `jobs`.
   - Ele usa **Ações Compostas** (os "Blocos") para agrupar `steps` repetitivos (ex: instalar Node, rodar Sonar).
   - O versionamento é feito usando **Git Tags** (ex: `v1`, `v2`).

2. **O Repositório "Consumidor" (ex: `my-app-nodejs`):**
   - Este é um dos 300+ repositórios de aplicação.
   - Seu arquivo de workflow é mínimo. Ele apenas "chama" o workflow centralizado, especificando a versão (tag) que deseja usar.

---

## 🌳 Estrutura dos Repositórios

```
my-org/
├── 📁 shared-workflows/ (O "Centro de Controle")
│   ├── 📁 .github/
│   │   └── 📁 workflows/
│   │       ├── 📄 reusable-node-ci.yml (Orquestrador v1: com Sonar)
│   │       └── 📄 reusable-node-v2.yml (Orquestrador v2: com Veracode)
│   └── 📁 actions/
│       ├── 📁 setup-node/
│       │   └── 📄 action.yml (Bloco: Instalar Node e dependências)
│       ├── 📁 run-sonar/
│       │   └── 📄 action.yml (Bloco: Rodar Sonar)
│       └── 📁 run-veracode/
│           └── 📄 action.yml (Bloco: Rodar Veracode)
│
└── 📁 my-app-nodejs/ (O "Consumidor" - uma das 300 apps)
    └── 📁 .github/
        └── 📁 workflows/
            └── 📄 ci.yml (O "Chamador" da app)
```

---

## 📄 Arquivos de Exemplo

Abaixo estão os conteúdos dos arquivos-chave que fazem essa arquitetura funcionar.

### 1. Os "Blocos" (Composite Actions)

Estes são os `steps` individuais, colocados no repositório `shared-workflows`.

#### `shared-workflows/actions/setup-node/action.yml`

```yaml
name: 'Setup Node.js Project'
description: 'Faz o checkout, instala o Node e as dependências'
runs:
  using: "composite"
  steps:
    - name: Checkout code
      uses: actions/checkout@v4

    - name: Setup Node.js
      uses: actions/setup-node@v4
      with:
        node-version: '18'
        cache: 'npm'

    - name: Install dependencies
      run: npm ci
      shell: bash
```

#### `shared-workflows/actions/run-sonar/action.yml`

```yaml
name: 'Run Sonar Scan'
description: 'Executa o SonarCloud Scan'
inputs:
  sonar-token:
    description: 'Token do Sonar'
    required: true
runs:
  using: "composite"
  steps:
    - name: SonarCloud Scan
      uses: SonarSource/sonarcloud-github-action@v2.1.1
      env:
        GITHUB_TOKEN: ${{ github.token }}
        SONAR_TOKEN: ${{ inputs.sonar-token }}
```

### 2. O "Orquestrador" (Reusable Workflow)

Este é o workflow reutilizável que define os jobs e chama os "blocos".

#### `shared-workflows/.github/workflows/reusable-node-ci.yml`

```yaml
name: Reusable Node.js CI
on:
  workflow_call:
    inputs:
      node-version:
        type: string
        required: true
    secrets:
      SONAR_TOKEN:
        required: true

jobs:
  build-and-test:
    name: Build and Test
    runs-on: ubuntu-latest
    steps:
      - name: Setup Node Project
        uses: my-org/shared-workflows/actions/setup-node@v1
        with:
          node-version: ${{ inputs.node-version }}

      - name: Run Unit Tests
        run: npm test

  security-scan:
    name: Security Scan
    runs-on: ubuntu-latest
    needs: build-and-test
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Run Sonar Scan
        uses: my-org/shared-workflows/actions/run-sonar@v1
        with:
          sonar-token: ${{ secrets.SONAR_TOKEN }}
```

### 3. O "Consumidor" (A Aplicação)

Este é o único arquivo necessário na sua aplicação de Node.js.

#### `my-app-nodejs/.github/workflows/ci.yml`

```yaml
name: CI Pipeline
on:
  push:
    branches: [ main ]
  pull_request:

jobs:
  build-and-scan:
    name: Run Main CI
    permissions:
      contents: read
    uses: my-org/shared-workflows/.github/workflows/reusable-node-ci.yml@v1
    with:
      node-version: '18'
    secrets:
      SONAR_TOKEN: ${{ secrets.SONAR_TOKEN_DA_APP }}
```

---

## 🔄 Gerenciando a Mudança (Sonar para Veracode)

O processo para migrar suas 300 apps é drasticamente simplificado:

**No `shared-workflows`:**

1. Crie o novo "bloco": `actions/run-veracode/action.yml`.
2. Crie um novo "orquestrador": `reusable-node-v2.yml`. Este arquivo será uma cópia do v1, mas chamará a action do Veracode em vez do Sonar.
3. Faça o commit, push e crie uma nova tag:
   ```bash
   git tag v2 && git push origin v2
   ```

**No `my-app-nodejs`:**

Simplesmente atualize o arquivo `ci.yml` para apontar para a nova versão:

```diff
- uses: my-org/shared-workflows/.github/workflows/reusable-node-ci.yml@v1
+ uses: my-org/shared-workflows/.github/workflows/reusable-node-v2.yml@v2
```

> **Nota:** Você também pode manter o mesmo nome de arquivo (ex: `reusable-node-ci.yml`) e apenas versioná-lo com tags. A v2 daquele arquivo conteria o Veracode.

Esta estratégia lhe dá controle centralizado, versionamento explícito e uma enorme redução de esforço de manutenção.
