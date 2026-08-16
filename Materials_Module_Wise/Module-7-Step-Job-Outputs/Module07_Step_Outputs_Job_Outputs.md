# Module 07 — Step Outputs and Job Outputs
### i27Academy — GitHub Actions Course

---

## Agenda

1. What are outputs and why we need them
2. Step Outputs
   - Setting a step output
   - Reading a step output
   - Multiple outputs from one step
3. Job Outputs
   - Setting a job output
   - Reading a job output
   - Using job output in conditions
4. $GITHUB_ENV vs $GITHUB_OUTPUT

---

## 1. What Are Outputs and Why We Need Them

In a workflow, jobs and steps often need to share data with each other.

We already know two ways to share data:

```
env:        → static values defined in the YAML file
$GITHUB_ENV → dynamic values shared between steps in the same job
```

But what if you need to pass a value from one job to another?

```
Job 1 generates an image tag → Job 2 needs that tag to build the image
Job 1 checks the branch      → Job 2 decides whether to deploy based on that
```

`$GITHUB_ENV` does not work across jobs — each job runs on a fresh machine.

This is where **outputs** come in:

```
Step outputs  → pass data between steps in the SAME job
Job outputs   → pass data between DIFFERENT jobs
```

---

## 2. Step Outputs

A step output lets you set a value in one step and read it in a later step within the same job.

### How to set a step output

```yaml
- name: My step
  id: my-step              # id is REQUIRED
  run: echo "key=value" >> $GITHUB_OUTPUT
```

### How to read a step output

```yaml
- name: Next step
  run: echo ${{ steps.my-step.outputs.key }}
#                  ↑ step id      ↑ key name
```

---

### 2.1 Basic Step Output

**07.1.1-step-output-basic.yml**

```yaml
name: 07.1.1-Step-Output-Basic

on:
  workflow_dispatch:

jobs:
  step-output-demo:
    runs-on: ubuntu-latest
    steps:
      - name: Set a step output
        id: my-step
        run: |
          echo "message=Hello from Step Output" >> $GITHUB_OUTPUT
          echo "Output has been set"

      - name: Read the step output
        run: |
          echo "============================="
          echo "READING STEP OUTPUT"
          echo "============================="
          echo "Message : ${{ steps.my-step.outputs.message }}"
          echo "============================="
```

Observe:
```
→ First step sets message via $GITHUB_OUTPUT
→ Second step reads it via steps.my-step.outputs.message
→ id: my-step is what connects the two steps
→ Without id: — you cannot reference the step
```

---

### 2.2 Multiple Outputs from One Step

**07.1.2-step-output-multiple.yml**

```yaml
name: 07.1.2-Step-Output-Multiple

on:
  workflow_dispatch:

jobs:
  multiple-outputs-demo:
    runs-on: ubuntu-latest
    steps:
      - name: Set multiple outputs
        id: build-info
        run: |
          echo "app-name=i27academy-devops-dashboard" >> $GITHUB_OUTPUT
          echo "image-tag=${{ github.sha }}" >> $GITHUB_OUTPUT
          echo "build-date=$(date +'%Y-%m-%d')" >> $GITHUB_OUTPUT
          echo "branch=${{ github.ref_name }}" >> $GITHUB_OUTPUT

      - name: Read all outputs
        run: |
          echo "App Name   : ${{ steps.build-info.outputs.app-name }}"
          echo "Image Tag  : ${{ steps.build-info.outputs.image-tag }}"
          echo "Build Date : ${{ steps.build-info.outputs.build-date }}"
          echo "Branch     : ${{ steps.build-info.outputs.branch }}"

      - name: Use output in docker tag
        run: |
          echo "Docker image would be:"
          echo "devopswithcloudhub/${{ steps.build-info.outputs.app-name }}:${{ steps.build-info.outputs.image-tag }}"
```

Observe:
```
→ One step sets four different outputs
→ All outputs share the same step id: build-info
→ Each output is a separate key=value line in $GITHUB_OUTPUT
→ All outputs are readable in later steps
```

Key points:
```
→ Step must have id: to be referenced
→ Set output: echo "key=value" >> $GITHUB_OUTPUT
→ Read output: ${{ steps.STEP_ID.outputs.KEY }}
→ Multiple outputs from one step — one line per output
→ Outputs are available from the next step onwards
→ Step outputs are scoped to the same job only
```

---

## 3. Job Outputs

A job output lets you pass a value from one job to another. It works in two parts:

**Part 1 — The job that sets the output:**
```yaml
jobs:
  my-job:
    outputs:                                        # declare job outputs
      my-key: ${{ steps.my-step.outputs.value }}   # points to a step output
    steps:
      - id: my-step
        run: echo "value=hello" >> $GITHUB_OUTPUT
```

**Part 2 — The job that reads the output:**
```yaml
  other-job:
    needs: my-job                                   # must depend on my-job
    steps:
      - run: echo ${{ needs.my-job.outputs.my-key }}
```

---

### 3.1 Basic Job Output

**07.2.1-job-output-basic.yml**

```yaml
name: 07.2.1-Job-Output-Basic

on:
  workflow_dispatch:

jobs:
  prepare:
    name: Prepare
    runs-on: ubuntu-latest
    outputs:
      image-tag: ${{ steps.tag.outputs.value }}
      app-name:  ${{ steps.app.outputs.value }}
    steps:
      - name: Generate image tag
        id: tag
        run: |
          echo "value=${{ github.sha }}" >> $GITHUB_OUTPUT

      - name: Set app name
        id: app
        run: |
          echo "value=i27academy-devops-dashboard" >> $GITHUB_OUTPUT

  build:
    name: Build
    runs-on: ubuntu-latest
    needs: prepare
    steps:
      - name: Use job outputs from prepare
        run: |
          echo "App Name  : ${{ needs.prepare.outputs.app-name }}"
          echo "Image Tag : ${{ needs.prepare.outputs.image-tag }}"
          echo "Full image: ${{ needs.prepare.outputs.app-name }}:${{ needs.prepare.outputs.image-tag }}"
```

Observe:
```
→ prepare job sets two outputs — image-tag and app-name
→ build job reads them via needs.prepare.outputs.*
→ needs: prepare is required — without it outputs are not accessible
→ Both jobs run on separate fresh VMs
→ Outputs are the bridge between them
```

---

### 3.2 Job Output in Conditions

**07.2.2-job-output-conditional.yml**

```yaml
name: 07.2.2-Job-Output-Conditional

on:
  workflow_dispatch:

jobs:
  check:
    name: Check Branch
    runs-on: ubuntu-latest
    outputs:
      should-deploy: ${{ steps.branch-check.outputs.value }}
      environment:   ${{ steps.env-check.outputs.value }}
    steps:
      - name: Check if this is main branch
        id: branch-check
        run: |
          if [ "${{ github.ref_name }}" = "main" ]; then
            echo "value=true" >> $GITHUB_OUTPUT
          else
            echo "value=false" >> $GITHUB_OUTPUT
          fi

      - name: Set environment based on branch
        id: env-check
        run: |
          if [ "${{ github.ref_name }}" = "main" ]; then
            echo "value=production" >> $GITHUB_OUTPUT
          else
            echo "value=development" >> $GITHUB_OUTPUT
          fi

  deploy:
    name: Deploy
    runs-on: ubuntu-latest
    needs: check
    if: needs.check.outputs.should-deploy == 'true'
    steps:
      - name: Deploy to environment
        run: |
          echo "Deploying to: ${{ needs.check.outputs.environment }}"

  summary:
    name: Summary
    runs-on: ubuntu-latest
    needs: check
    if: always()
    steps:
      - name: Print summary
        run: |
          echo "Should Deploy : ${{ needs.check.outputs.should-deploy }}"
          echo "Environment   : ${{ needs.check.outputs.environment }}"
```

Observe:
```
→ check job computes should-deploy and environment
→ deploy job uses should-deploy in its if: condition
→ Run from main branch → deploy job runs
→ Run from any other branch → deploy job is skipped
→ summary job always runs regardless
```

Key points:
```
→ Job must declare outputs: block
→ outputs: block points to step outputs
→ Reading job must declare needs:
→ Read via: ${{ needs.JOB_ID.outputs.KEY }}
→ Outputs are always strings — compare with == 'true' not == true
→ Jenkins equivalent: env vars shared between stages
```

---

## 4. $GITHUB_ENV vs $GITHUB_OUTPUT

This is one of the most commonly confused concepts in GitHub Actions.

**07.3.1-github-env-vs-output.yml**

```yaml
name: 07.3.1-GITHUB-ENV-vs-OUTPUT

on:
  workflow_dispatch:

jobs:
  job-one:
    name: Job One
    runs-on: ubuntu-latest
    outputs:
      output-value: ${{ steps.set-output.outputs.value }}
    steps:
      - name: Set via GITHUB_ENV
        run: |
          echo "ENV_VALUE=set-via-github-env" >> $GITHUB_ENV

      - name: Read GITHUB_ENV in same job
        run: |
          echo "ENV_VALUE is: $ENV_VALUE"    # works — same job

      - name: Set via GITHUB_OUTPUT
        id: set-output
        run: |
          echo "value=set-via-github-output" >> $GITHUB_OUTPUT

  job-two:
    name: Job Two
    runs-on: ubuntu-latest
    needs: job-one
    steps:
      - name: Try to read GITHUB_ENV from job-one
        run: |
          echo "ENV_VALUE from job-one : $ENV_VALUE"
          echo "Above is EMPTY — GITHUB_ENV does not cross jobs"

      - name: Read GITHUB_OUTPUT from job-one
        run: |
          echo "Output from job-one : ${{ needs.job-one.outputs.output-value }}"
          echo "Above works — GITHUB_OUTPUT crosses jobs via outputs:"
```

Observe:
```
→ job-one sets ENV_VALUE via $GITHUB_ENV
→ job-one reads ENV_VALUE — works ✅
→ job-two tries to read ENV_VALUE — EMPTY ❌
→ job-two reads output-value via needs.job-one.outputs — works ✅
```

**The clear rule:**

| | $GITHUB_ENV | $GITHUB_OUTPUT |
|---|---|---|
| Syntax | `echo "KEY=value" >> $GITHUB_ENV` | `echo "key=value" >> $GITHUB_OUTPUT` |
| Available from | Next step in same job | Other jobs via `needs.*.outputs.*` |
| Scope | Same job only | Across jobs |
| Requires | Nothing extra | `id:` on step + `outputs:` on job |
| Use for | Sharing data within a job | Sharing data between jobs |

```
$GITHUB_ENV    = talking to steps in THIS job
$GITHUB_OUTPUT = talking to OTHER jobs
```

---

## Module 07 Summary

- Step outputs pass data between steps in the same job
- Set a step output: `echo "key=value" >> $GITHUB_OUTPUT`
- Read a step output: `${{ steps.STEP_ID.outputs.KEY }}`
- `id:` on a step is required to reference it
- Job outputs pass data between different jobs
- Job must declare `outputs:` block pointing to step outputs
- Reading job must declare `needs:` on the job that set the output
- Read a job output: `${{ needs.JOB_ID.outputs.KEY }}`
- Job outputs are always strings — compare with `== 'true'` not `== true`
- `$GITHUB_ENV` — shares data between steps in the same job
- `$GITHUB_OUTPUT` — shares data between jobs via `outputs:` and `needs:`

---

## File Summary

| File | What it demonstrates |
|---|---|
| 07.1.1-step-output-basic.yml | Set and read a single step output |
| 07.1.2-step-output-multiple.yml | Multiple outputs from one step |
| 07.2.1-job-output-basic.yml | Pass output from one job to another |
| 07.2.2-job-output-conditional.yml | Use job output in if: condition |
| 07.3.1-github-env-vs-output.yml | Clear comparison of both approaches |

---

*i27Academy · GitHub Actions Course · Module 07 · i27academy.com*
