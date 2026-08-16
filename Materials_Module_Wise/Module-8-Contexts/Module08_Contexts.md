# Module 08 — Contexts
### i27Academy — GitHub Actions Course

---

## Agenda

1. What is a context
2. github.*
3. runner.*
4. env.*
5. vars.*
6. secrets.*
7. steps.*
8. needs.*
9. inputs.*
10. matrix.*
11. Debugging contexts with toJson()

---

## 1. What is a Context

A context is an object that GitHub Actions provides automatically at runtime. It contains information about the workflow run, the repository, the runner, the inputs and more.

You access context values using the expression syntax:

```
${{ context.property }}
  ↑         ↑
  │         └── property name
  └── context name
```

GitHub provides 9 contexts. Each gives you a different category of information:

```
github.*   → run + repo + event information
runner.*   → the machine running the job
env.*      → environment variables
vars.*     → repository variables from GitHub UI
secrets.*  → encrypted secrets
steps.*    → outputs from previous steps in the same job
needs.*    → outputs and results from previous jobs
inputs.*   → values passed via workflow_dispatch or workflow_call
matrix.*   → current matrix combination (covered in Module 11)
```

---

## 2. github.*

The most commonly used context. Contains information about the current run, repository, event and actor.

**08.1.1-context-github.yml**

```yaml
name: 08.1.1-Context-Github

on:
  workflow_dispatch:

jobs:
  github-context-demo:
    runs-on: ubuntu-latest
    steps:
      - name: Print github.* context
        run: |
          echo "Repository     : ${{ github.repository }}"
          echo "Ref Name       : ${{ github.ref_name }}"
          echo "Ref            : ${{ github.ref }}"
          echo "SHA            : ${{ github.sha }}"
          echo "Actor          : ${{ github.actor }}"
          echo "Event Name     : ${{ github.event_name }}"
          echo "Workflow       : ${{ github.workflow }}"
          echo "Job            : ${{ github.job }}"
          echo "Run ID         : ${{ github.run_id }}"
          echo "Run Number     : ${{ github.run_number }}"
          echo "Workspace      : ${{ github.workspace }}"

      - name: Dump full github context
        env:
          GITHUB_CONTEXT: ${{ toJson(github) }}
        run: echo "$GITHUB_CONTEXT"
```

**Most used github.* properties:**

| Property | What it returns | Example |
|---|---|---|
| `github.repository` | owner/repo name | i27academy/devops-dashboard |
| `github.ref_name` | branch or tag name | main |
| `github.ref` | full ref path | refs/heads/main |
| `github.sha` | full commit SHA | abc123def456... |
| `github.actor` | who triggered the run | siva |
| `github.event_name` | what triggered it | push, pull_request |
| `github.run_number` | run count for this workflow | 42 |
| `github.run_id` | unique ID for this run | 1234567890 |
| `github.workflow` | workflow name | CI Pipeline |
| `github.workspace` | path to checked out code | /home/runner/work/... |

---

## 3. runner.*

Contains information about the machine running the current job.

**08.1.2-context-runner.yml**

```yaml
name: 08.1.2-Context-Runner

on:
  workflow_dispatch:

jobs:
  runner-context-demo:
    runs-on: ubuntu-latest
    steps:
      - name: Print runner.* context
        run: |
          echo "OS         : ${{ runner.os }}"
          echo "Arch       : ${{ runner.arch }}"
          echo "Name       : ${{ runner.name }}"
          echo "Temp       : ${{ runner.temp }}"
          echo "Tool Cache : ${{ runner.tool_cache }}"

      - name: Conditional step based on OS
        if: runner.os == 'Linux'
        run: echo "This step only runs on Linux ✅"
```

**runner.* properties:**

| Property | What it returns | Example |
|---|---|---|
| `runner.os` | Operating system | Linux, Windows, macOS |
| `runner.arch` | CPU architecture | X64, ARM64 |
| `runner.name` | Name of the runner | GitHub Actions 2 |
| `runner.temp` | Path to temp directory | /home/runner/work/_temp |
| `runner.tool_cache` | Path to tool cache | /opt/hostedtoolcache |

---

## 4. env.*

Access environment variables set via `env:` anywhere in the workflow. Useful when you need to reference an env var outside a `run:` block.

```yaml
env:
  APP_NAME: i27academy-devops-dashboard

jobs:
  demo:
    steps:
      - name: Use env.* in a condition
        if: env.APP_NAME == 'i27academy-devops-dashboard'
        run: echo "Matched ✅"

      - name: Use env.* in with:
        uses: actions/setup-java@v4
        with:
          java-version: ${{ env.JAVA_VERSION }}
```

Key point:
```
Inside run:      → use $APP_NAME (shell syntax)
Outside run:     → use ${{ env.APP_NAME }} (expression syntax)
  (in with:, if:, name:)
```

---

## 5. vars.*

Access repository, environment or organization variables set in GitHub UI Settings.

```yaml
steps:
  - name: Use vars.*
    run: |
      echo "Deploy Env : ${{ vars.DEPLOY_ENV }}"
      echo "App Name   : ${{ vars.APP_NAME }}"
```

Key point:
```
→ vars.* values are visible in logs — not masked
→ Use for non-sensitive config only
→ Set in: Repo → Settings → Secrets and variables → Variables
```

---

## 6. secrets.*

Access encrypted secrets. Values are always masked in logs.

```yaml
steps:
  - name: Use secrets.*
    run: |
      echo "Secret     : ${{ secrets.MY_SECRET }}"
      echo "Is set     : ${{ secrets.MY_SECRET != '' }}"
      echo "Token set  : ${{ secrets.GITHUB_TOKEN != '' }}"
```

Key points:
```
→ secrets.* values are always masked — show as ***
→ GITHUB_TOKEN is auto-created every run — no setup needed
→ Environment secrets only available when environment: is declared
```

**08.1.3-context-env-vars-secrets.yml** — demonstrates all three together.

---

## 7. steps.*

Access outputs and results from previous steps in the same job.

```yaml
steps:
  - name: Set output
    id: my-step
    run: echo "value=hello" >> $GITHUB_OUTPUT

  - name: Read output
    run: |
      echo "Output  : ${{ steps.my-step.outputs.value }}"
      echo "Outcome : ${{ steps.my-step.outcome }}"
```

**steps.* properties:**

| Property | What it returns |
|---|---|
| `steps.ID.outputs.KEY` | Output value set via $GITHUB_OUTPUT |
| `steps.ID.outcome` | Result — success, failure, skipped, cancelled |
| `steps.ID.conclusion` | Final result after continue-on-error |

**08.1.4-context-steps-needs.yml** — demonstrates steps.* and needs.* together.

---

## 8. needs.*

Access outputs and results from previous jobs.

```yaml
jobs:
  job-one:
    outputs:
      result: ${{ steps.compute.outputs.value }}
    steps:
      - id: compute
        run: echo "value=hello" >> $GITHUB_OUTPUT

  job-two:
    needs: job-one
    steps:
      - run: |
          echo "Result : ${{ needs.job-one.result }}"
          echo "Output : ${{ needs.job-one.outputs.result }}"
```

**needs.* properties:**

| Property | What it returns |
|---|---|
| `needs.JOB.result` | Job result — success, failure, skipped, cancelled |
| `needs.JOB.outputs.KEY` | Job output value |

---

## 9. inputs.*

Access values passed to the workflow via `workflow_dispatch` or `workflow_call`.

**08.1.5-context-inputs.yml**

```yaml
name: 08.1.5-Context-Inputs

on:
  workflow_dispatch:
    inputs:
      environment:
        description: Target environment
        type: choice
        options: [ development, staging, production ]
        default: development
      version:
        description: Version to deploy
        type: string
        default: 'v1.0.0'
      skip-tests:
        description: Skip tests?
        type: boolean
        default: false

jobs:
  inputs-context-demo:
    runs-on: ubuntu-latest
    steps:
      - name: Print inputs.* context
        run: |
          echo "Environment : ${{ inputs.environment }}"
          echo "Version     : ${{ inputs.version }}"
          echo "Skip Tests  : ${{ inputs.skip-tests }}"

      - name: Use inputs in conditions
        if: inputs.environment == 'production'
        run: echo "Running production steps ✅"

      - name: Skip tests if requested
        if: inputs.skip-tests == false
        run: echo "Running tests — skip-tests is false"
```

Key points:
```
→ inputs.* only available on workflow_dispatch or workflow_call
→ Access as ${{ inputs.INPUT_NAME }}
→ Type is preserved — string, boolean, choice
→ boolean inputs compare as: inputs.skip-tests == false
```

---

## 9. inputs.*

Access values passed to the workflow via `workflow_dispatch`. This allows users to provide values when manually triggering a workflow.

### Defining inputs

```yaml
on:
  workflow_dispatch:
    inputs:
      INPUT_NAME:
        description: What this input is for  # shown in GitHub UI
        required: true                        # true = must provide
        type: string                          # input type
        default: 'default-value'             # used when not provided
```

### Four input types

**string — free text:**
```yaml
version:
  description: Version to deploy
  type: string
  default: 'v1.0.0'
```

**boolean — true/false checkbox:**
```yaml
skip-tests:
  description: Skip tests?
  type: boolean
  default: false
```

**choice — dropdown with predefined options:**
```yaml
environment:
  description: Target environment
  type: choice
  options:
    - development
    - staging
    - production
  default: development
```

**number — numeric value:**
```yaml
replicas:
  description: Number of replicas
  type: number
  default: 1
```

### Accessing inputs

```yaml
${{ inputs.environment }}     # choice or string
${{ inputs.version }}         # string
${{ inputs.skip-tests }}      # boolean
${{ inputs.replicas }}        # number
```

**08.1.6-inputs-workflow-dispatch.yml**

```yaml
name: 08.1.6-Inputs-Workflow-Dispatch

on:
  workflow_dispatch:
    inputs:
      environment:
        description: Target environment to deploy
        required: true
        type: choice
        options:
          - development
          - staging
          - production
        default: development
      version:
        description: Version to deploy (e.g. v1.0.0)
        required: false
        type: string
        default: 'v1.0.0'
      skip-tests:
        description: Skip tests? Use only in emergencies
        required: false
        type: boolean
        default: false
      replicas:
        description: Number of replicas to deploy
        required: false
        type: number
        default: 1

jobs:
  inputs-demo:
    runs-on: ubuntu-latest
    steps:
      - name: Print all inputs
        run: |
          echo "Environment : ${{ inputs.environment }}"
          echo "Version     : ${{ inputs.version }}"
          echo "Skip Tests  : ${{ inputs.skip-tests }}"
          echo "Replicas    : ${{ inputs.replicas }}"

      - name: Use choice input in condition
        if: inputs.environment == 'production'
        run: echo "Production deployment ✅"

      - name: Use boolean input in condition
        if: inputs.skip-tests == false
        run: echo "Running tests ✅"

      - name: Use string input
        run: echo "Deploying version ${{ inputs.version }}"

      - name: Use number input
        run: echo "Deploying ${{ inputs.replicas }} replica(s)"
```

Observe:
```
→ Run workflow button shows all inputs with descriptions
→ environment shows as a dropdown
→ skip-tests shows as a checkbox
→ version and replicas show as text fields
→ All inputs accessible via ${{ inputs.INPUT_NAME }}
```

Key points:
```
→ Inputs are only available on workflow_dispatch (and workflow_call — Module 14)
→ required: true — user must provide a value — no default used
→ required: false — default value is used if not provided
→ boolean inputs compare as: inputs.skip-tests == false
→ choice and string compare with quotes: inputs.environment == 'production'
→ inputs.* context — covered in Module 08 contexts
→ workflow_call inputs covered in Module 14 — Creating Reusable Workflows
```

---

## 10. matrix.*

Access the current matrix combination values. Only available in matrix jobs.

```yaml
jobs:
  test:
    strategy:
      matrix:
        java-version: [ '17', '21' ]
        os: [ ubuntu-latest, windows-latest ]
    runs-on: ${{ matrix.os }}
    steps:
      - run: |
          echo "Java : ${{ matrix.java-version }}"
          echo "OS   : ${{ matrix.os }}"
```

We cover matrix.* in full detail in **Module 11 — Working with Matrices**.

---

## 11. Debugging Contexts with toJson()

When you are not sure what is available in a context, dump the entire thing as JSON.

**Always assign toJson() to an env var first — never put it directly in run:**

```yaml
# ✅ Correct pattern
- name: Dump github context
  env:
    GITHUB_CONTEXT: ${{ toJson(github) }}
  run: echo "$GITHUB_CONTEXT"

# ❌ Can cause YAML issues
- name: Dump github context
  run: echo "${{ toJson(github) }}"
```

**08.2.1-all-contexts-demo.yml** — prints all available contexts in one workflow.

---

## All Contexts Quick Reference

| Context | What it contains | Common properties |
|---|---|---|
| `github.*` | Run + repo + event info | sha, ref_name, actor, event_name, run_number |
| `runner.*` | Machine information | os, arch, name, temp |
| `env.*` | Workflow env variables | any key defined in env: |
| `vars.*` | GitHub UI variables | any variable from Settings |
| `secrets.*` | Encrypted secrets | any secret + GITHUB_TOKEN |
| `steps.*` | Previous step outputs | steps.ID.outputs.KEY, steps.ID.outcome |
| `needs.*` | Previous job outputs | needs.JOB.outputs.KEY, needs.JOB.result |
| `inputs.*` | Workflow dispatch inputs | inputs.INPUT_NAME |
| `matrix.*` | Current matrix values | matrix.java-version, matrix.os |

---

## Module 08 Summary

- A context is an object GitHub provides automatically at runtime
- Access via expression syntax: `${{ context.property }}`
- `github.*` — most used — run, repo and event information
- `runner.*` — machine information — OS, arch, name
- `env.*` — workflow env vars — use in conditions and with: inputs
- `vars.*` — GitHub UI variables — plain text, visible in logs
- `secrets.*` — encrypted secrets — always masked in logs
- `steps.*` — previous step outputs and outcomes — same job only
- `needs.*` — previous job outputs and results — across jobs
- `inputs.*` — dispatch or call inputs — only on workflow_dispatch/call
- `matrix.*` — current matrix combination — covered in Module 11
- Debug any context with `toJson()` — always assign to env var first

---

## File Summary

| File | What it demonstrates |
|---|---|
| 08.1.1-context-github.yml | github.* properties + full JSON dump |
| 08.1.2-context-runner.yml | runner.* properties + OS condition |
| 08.1.3-context-env-vars-secrets.yml | env.*, vars.*, secrets.* together |
| 08.1.4-context-steps-needs.yml | steps.* and needs.* across jobs |
| 08.1.5-context-inputs.yml | inputs.* context usage |
| 08.1.6-inputs-workflow-dispatch.yml | All input types — string, boolean, choice, number |
| 08.2.1-all-contexts-demo.yml | All contexts in one workflow |

---

*i27Academy · GitHub Actions Course · Module 08 · i27academy.com*