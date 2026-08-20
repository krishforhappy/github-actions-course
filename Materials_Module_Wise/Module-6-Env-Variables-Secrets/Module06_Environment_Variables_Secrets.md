# Module 06 — Environment Variables & Secrets
### i27Academy — GitHub Actions Course

---

## Agenda

1. Environment Variables (env:)
   - Workflow level
   - Job level
   - Step level
   - Variable precedence
2. Dynamic Variables ($GITHUB_ENV)
   - Basic usage
   - Build versioning
   - Docker image tagging
   - Dynamic environment based on branch
3. Repository Variables (vars.*)
4. Secrets (secrets.*)
   - Repository secrets
   - Environment secrets
   - Organization secrets
   - GITHUB_TOKEN
   - Secrets masking

---

## 1. Environment Variables (env:)

Environment variables allow you to store values and reference them throughout your workflow. They are defined using the `env:` key and can be set at three levels — workflow, job, or step.

---

### 1.1 Workflow Level

Variables defined at the top of the workflow file are available to every job and every step.

**06.1.1-env-workflow-level.yml**

```yaml
name: 01-env-variables.yaml
on:
  workflow_dispatch:
env:
  APP_NAME: i27Academy
  ENVIRONMENT: Dev
jobs:
  env-demo:
    runs-on: ubuntu-latest
    steps:
    - name: Print Variables
      run: |
        echo "Printing environment Variables"
        echo "Application Name is: $APP_NAME"
        echo "Environment Name is: $ENVIRONMENT"
```

Observe:
```
→ APP_NAME and ENVIRONMENT are accessible from any job or step
→ Values print as defined — plain text, visible in logs
→ Use workflow level for values that never change across jobs
   e.g. app name, registry, project name
```

---

### 1.2 Job Level

Variables defined under a job's `env:` are scoped to that job only. Different jobs can define the same variable name with different values.

**06.1.2-env-job-level.yml**

```yaml
name: 02-Job-Level-Variables

on:
  workflow_dispatch:

jobs:
  dev-job:
    runs-on: ubuntu-latest
    env:
      ENVIRONMENT: dev
      APP_NAME: i27Academy-dev-app
    steps:
      - name: Print Variables
        run: |
          echo "Application : $APP_NAME"
          echo "Environment : $ENVIRONMENT"

  prod-job:
    runs-on: ubuntu-latest
    env:
      ENVIRONMENT: prod
      APP_NAME: i27Academy-prod-app
    steps:
      - name: Print Variables
        run: |
          echo "Application : $APP_NAME"
          echo "Environment : $ENVIRONMENT"
```

Observe:
```
→ dev-job prints: i27Academy-dev-app / dev
→ prod-job prints: i27Academy-prod-app / prod
→ Same variable names — different values per job
→ Job level variables do not leak between jobs
```

---

### 1.3 Step Level

Variables defined under a step's `env:` are scoped to that single step only. They are not visible to any other step in the same job.

**06.1.3-env-step-level.yml**

```yaml
name: 03-step-level-env
on: workflow_dispatch
jobs:
  dev-job:
    runs-on: ubuntu-latest
    steps:
      - name: Print Variables
        env:
          ENVIRONMENT: dev
          APP_NAME: helpdesk
        run: |
          echo "Application Name is: $APP_NAME"
          echo "Environment is: $ENVIRONMENT"
      - name: Print Variables Other
        run: |
          echo "Application Name is: $APP_NAME"
          echo "Environment is: $ENVIRONMENT"
```

Observe:
```
→ First step prints: helpdesk / dev
→ Second step prints nothing — variables are gone
→ Step level env is the most restrictive scope
→ Use when a variable is only needed for one specific step
```

---

### 1.4 Variable Precedence

When the same variable name is defined at multiple levels, the most specific level wins.

```
Step level    ← highest priority — overrides everything
     ↓
Job level     ← overrides workflow level
     ↓
Workflow level ← lowest priority
```

**06.1.4-env-variable-precedence.yml**

```yaml
name: 04-variable-precedence
on: workflow_dispatch
env:
  APPLICATION_NAME: i27-app-workflow
jobs:
  dev-job:
    runs-on: ubuntu-latest
    env:
      APPLICATION_NAME: i27-app-job-level
    steps:
      - name: Print Env Variables
        env:
          APPLICATION_NAME: i27-app-step-level
        run:
          echo "Application name is: $APPLICATION_NAME"
```

Observe:
```
→ Step level wins → prints: i27-app-step-level
→ If step level is removed → job level wins → i27-app-job-level
→ If job level is also removed → workflow level → i27-app-workflow
```

---

### env: vs ${{ env.VAR }} — which to use

```yaml
# Inside run: — use shell syntax (simpler)
run: echo $APP_NAME

# Outside run: (in with:, if:, name:) — use expression syntax (required)
- uses: actions/setup-java@v4
  with:
    java-version: ${{ env.JAVA_VERSION }}

- name: Deploy to ${{ env.ENVIRONMENT }}
  if: env.ENVIRONMENT == 'production'
```

Key points:
```
→ workflow level → available to all jobs and steps
→ job level     → available to all steps in that job only
→ step level    → available to that step only
→ step > job > workflow — most specific wins
→ env: values are static — defined before the workflow runs
→ Jenkins equivalent: environment {} block in Jenkinsfile
```

---

## 2. Dynamic Variables ($GITHUB_ENV)

`$GITHUB_ENV` is a special file provided by GitHub Actions that lets you set environment variables at runtime — during the workflow execution — and make them available to subsequent steps in the same job.

**Key difference from `env:`:**

```
env:        → static — values are known before the workflow starts
$GITHUB_ENV → dynamic — values are computed during the workflow run
```

---

### 2.1 Basic Usage

**06.2.1-github-env-basic.yml**

```yaml
name: 05-github-env
on: workflow_dispatch
jobs:
  github-env-demo:
    runs-on: ubuntu-latest
    steps:
    - name: Set Variable
      run: |
        echo "APP_VERSION=1.0" >> $GITHUB_ENV
        echo "NAME=Siva" >> $GITHUB_ENV
        echo "Trying to print APP_VERSION in the same step"
        echo $APP_VERSION
        echo $NAME

    - name: Use Variable
      run: |
        echo "Application Version is $APP_VERSION"
        echo "Name is: $NAME"
```

Observe:
```
→ First step — APP_VERSION and NAME print nothing (same step)
→ Second step — prints 1.0 and Siva correctly
→ Variable is available ONLY from the next step onwards
→ Not available in the same step that wrote it
```

---

### 2.2 Build Versioning

**06.2.2-github-env-build.yml**

```yaml
name: 06-github-env-build-example
on: workflow_dispatch
jobs:
  build-env-job:
    runs-on: ubuntu-latest
    steps:
    - name: Set Env variables
      run: |
        echo "BUILD_VERSION_ID=${{ github.run_id }}" >> $GITHUB_ENV
        echo "BUILD_VERSION_NUMBER=${{ github.run_number }}" >> $GITHUB_ENV
    - name: Use build version
      run: |
        echo "Build Version ID is: $BUILD_VERSION_ID"
        echo "Build Version Number is: $BUILD_VERSION_NUMBER"

  second-job:
    runs-on: ubuntu-latest
    steps:
    - name: Print GITHUB_ENV from different job
      run: |
        echo "Build Version ID is: $BUILD_VERSION_ID"
        echo "Build Version Number is: $BUILD_VERSION_NUMBER"
```

Observe:
```
→ build-env-job → BUILD_VERSION_ID and BUILD_VERSION_NUMBER print correctly
→ second-job   → both values print EMPTY
→ $GITHUB_ENV is scoped to the same job — does NOT carry to other jobs
→ For cross-job data use job outputs (covered in Module 07)
```

---

### 2.3 Docker Image Tagging

**06.2.3-github-env-docker.yml**

```yaml
name: 07-github-env-docker-example
on: workflow_dispatch
jobs:
  create_docker_image_job:
    name: Create Docker Tag Job
    runs-on: ubuntu-latest
    steps:
    - name: Create Docker Tag
      run: |
        echo "IMAGE_TAG=${{ github.sha }}" >> $GITHUB_ENV
    - name: Build Docker Image
      run: |
        echo "Docker Build Image is: i27academy:$IMAGE_TAG"
```

Observe:
```
→ IMAGE_TAG is set once using the commit SHA
→ Reused in the build step — consistent, traceable tag
→ github.sha guarantees a unique tag per commit
→ This is the real-world pattern for Docker image versioning
```

---

### 2.4 Dynamic Environment Based on Branch

**06.2.4-github-env-dynamic.yml**

```yaml
name: 08-Dynamic-Env
on: workflow_dispatch
jobs:
  print-job:
    name: Print Job Based on Branch
    runs-on: ubuntu-latest
    steps:
    - name: Determine the Env
      run: |
        if [ "${{ github.ref_name }}" = "main" ]; then
          echo "BranchName=main" >> $GITHUB_ENV
        else
          echo "BranchName=Other" >> $GITHUB_ENV
        fi
    - name: Print Environment
      run: |
        echo "Branch Deployed is $BranchName Branch"
```

Observe:
```
→ Shell if/else computes the value at runtime
→ Result is written to $GITHUB_ENV once
→ All subsequent steps can use $BranchName
→ No need to repeat the if/else logic in every step
→ Foundation of branch-based deployment pipelines
```

Key points:
```
→ Write to $GITHUB_ENV: echo "KEY=value" >> $GITHUB_ENV
→ Variable available from the NEXT step onwards — not the same step
→ Scoped to the same job — does not cross to other jobs
→ Use for: docker tags, build numbers, computed deploy targets
→ Windows: "KEY=value" >> $env:GITHUB_ENV
```

---

## 3. Repository Variables (vars.*)

Repository variables are non-sensitive configuration values stored in GitHub UI — not in your YAML file. They can be changed without editing any workflow file.

**Setup:**
```
Repo → Settings → Secrets and variables → Actions → Variables tab
→ New repository variable
  APP_NAME   = i27Academy
  DEPLOY_ENV = development
  REGISTRY   = docker.io
```

**06.3.1-repository-variables.yml**

```yaml
name: 06.3.1-Repository-Variables

on:
  workflow_dispatch:

jobs:
  variables-demo:
    runs-on: ubuntu-latest
    steps:
      - name: Print repository variables
        run: |
          echo "============================="
          echo "REPOSITORY VARIABLES (vars.*)"
          echo "============================="
          echo "APP_NAME   : ${{ vars.APP_NAME }}"
          echo "DEPLOY_ENV : ${{ vars.DEPLOY_ENV }}"
          echo "REGISTRY   : ${{ vars.REGISTRY }}"
          echo "============================="
          echo "Note: values ARE visible in logs — not masked"

      - name: Compare env vs vars
        env:
          ENV_VAR: "I am defined in the YAML file"
        run: |
          echo "env var  : $ENV_VAR"
          echo "vars.*   : ${{ vars.APP_NAME }}"
```

Observe:
```
→ vars.* values print as plain text — not masked
→ Use vars.* for non-sensitive config only
→ Change the value in GitHub UI — no YAML change needed
```

**Three levels of variables (same as secrets):**

```
Repository variables  → all workflows in this repo
Environment variables → only jobs with matching environment:
Organization variables → shared across multiple repos
```

**env: vs vars.*:**

```
env:    → defined in the YAML file
         change = edit the file + commit + push

vars.*  → defined in GitHub UI Settings
         change = update in UI — no code change needed
         useful for: cluster names, regions, app versions, URLs
```

---

## 4. Secrets (secrets.*)

Secrets are encrypted values stored by GitHub. They are never visible in plain text — not in the UI, not in logs, not to anyone.

**Setup:**
```
Repo → Settings → Secrets and variables → Actions → Secrets tab
→ New repository secret
  MY_SECRET = any-sensitive-value
```

**06.4.1-secrets-demo.yml**

```yaml
name: 06.4.1-Secrets-Demo

on:
  workflow_dispatch:

jobs:
  secrets-demo:
    runs-on: ubuntu-latest
    steps:
      - name: Secrets are masked in logs
        run: |
          echo "MY_SECRET : ${{ secrets.MY_SECRET }}"
          echo "Above line shows *** in logs — never the real value"

      - name: Safe pattern — assign to env var first
        env:
          MY_SECRET: ${{ secrets.MY_SECRET }}
        run: |
          echo "Secret length : ${#MY_SECRET}"
          echo "Secret is set : ${{ secrets.MY_SECRET != '' }}"

      - name: GITHUB_TOKEN — auto created every run
        run: |
          echo "GITHUB_TOKEN is set : ${{ secrets.GITHUB_TOKEN != '' }}"
          echo "Auto-created for every workflow run — no setup needed"

  environment-secrets-demo:
    runs-on: ubuntu-latest
    environment: staging
    steps:
      - name: Environment secrets
        run: |
          echo "This job targets the staging environment"
          echo "Environment secrets are only accessible"
          echo "when environment: is declared on the job"
```

Observe:
```
→ secrets.MY_SECRET prints *** in logs — masked automatically
→ Safe pattern: assign to env var, check length or is-set
→ GITHUB_TOKEN is always available — no setup needed
→ environment-secrets-demo job targets staging environment
→ environment secrets only unlock when environment: is declared
```

---

### Three levels of secrets

```
Repository secrets
  → Repo → Settings → Secrets → Actions
  → Available to all workflows in this repo

Environment secrets
  → Repo → Settings → Environments → [name] → Secrets
  → Only available when job declares environment: [name]
  → If job has no environment: declared → secret is silently empty

Organization secrets
  → Org → Settings → Secrets → Actions
  → Shared across selected repos in the org
  → Useful for: DOCKER_USERNAME, DOCKER_PASSWORD across all services
```

---

### GITHUB_TOKEN

GitHub automatically creates a `GITHUB_TOKEN` secret for every workflow run. No setup required.

```yaml
- name: Use GITHUB_TOKEN
  run: |
    echo "Token is set: ${{ secrets.GITHUB_TOKEN != '' }}"
```

What it can do:
```
→ Read and write repository contents
→ Create releases
→ Comment on pull requests and issues
→ Trigger other workflows
→ Push to the repo from within a workflow
```

Permissions can be restricted at workflow or job level — covered in **Module 18 — Security Hardening**.

---

### Secrets masking

GitHub automatically replaces secret values with `***` in logs — even if you accidentally print them.

```yaml
- run: echo "${{ secrets.MY_SECRET }}"
# prints: ***
```

**Safe pattern — never print secrets directly:**

```yaml
- name: Check if secret is set
  env:
    MY_SECRET: ${{ secrets.MY_SECRET }}
  run: |
    echo "Length : ${#MY_SECRET}"        # prints length — not the value
    echo "Is set : ${{ secrets.MY_SECRET != '' }}"  # prints true/false
```

---

## Summary — All Variable Types

| Type | Where defined | Visible in logs | Scope | Use for |
|---|---|---|---|---|
| `env:` (workflow) | YAML file | Yes | All jobs + steps | Static config for whole workflow |
| `env:` (job) | YAML file | Yes | That job only | Job-specific config |
| `env:` (step) | YAML file | Yes | That step only | Step-specific config |
| `$GITHUB_ENV` | Runtime (shell) | Yes | Same job, next steps | Dynamic computed values |
| `vars.*` | GitHub UI | Yes | Repo / Env / Org | Non-sensitive config |
| `secrets.*` | GitHub UI (encrypted) | No (masked) | Repo / Env / Org | Passwords, tokens, keys |

---

## Module 06 Summary

- `env:` defines static environment variables at workflow, job, or step level
- Step level overrides job level overrides workflow level — most specific wins
- `$GITHUB_ENV` sets dynamic variables at runtime — available from the next step onwards
- `$GITHUB_ENV` variables are scoped to the same job — they do not cross job boundaries
- `vars.*` stores non-sensitive config in GitHub UI — change without touching YAML
- `secrets.*` stores encrypted values — always masked in logs
- Three levels for both vars and secrets — repository, environment, organization
- Environment secrets are silently empty without `environment:` declared on the job
- `GITHUB_TOKEN` is auto-created for every run — no setup needed
- For cross-job data use job outputs — covered in Module 07

---

## File Summary

| File | What it demonstrates |
|---|---|
| 06.1.1-env-workflow-level.yml | env: at workflow level |
| 06.1.2-env-job-level.yml | env: at job level — different values per job |
| 06.1.3-env-step-level.yml | env: at step level — not visible to other steps |
| 06.1.4-env-variable-precedence.yml | step > job > workflow precedence |
| 06.2.1-github-env-basic.yml | $GITHUB_ENV basic usage — set and read |
| 06.2.2-github-env-build.yml | Build versioning + cross-job limitation |
| 06.2.3-github-env-docker.yml | Docker image tagging with github.sha |
| 06.2.4-github-env-dynamic.yml | Branch-based dynamic environment |
| 06.3.1-repository-variables.yml | vars.* from GitHub UI Settings |
| 06.4.1-secrets-demo.yml | secrets.*, masking, GITHUB_TOKEN |

---

*i27Academy · GitHub Actions Course · Module 06 · i27academy.com*
