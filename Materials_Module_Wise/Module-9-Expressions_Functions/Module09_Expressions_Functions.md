# Module 09 — Expressions & Functions
### i27Academy — GitHub Actions Course

---

## Agenda

1. Expression syntax
2. Operators — comparison and logical
3. String functions — contains, startsWith, endsWith, format, join
4. Data functions — toJson, fromJson, hashFiles
5. Status check functions — success, failure, always, cancelled
6. if: conditions — real world patterns

---

## 1. Expression Syntax

Expressions in GitHub Actions are written inside `${{ }}`. GitHub evaluates what is inside and returns a result.

```yaml
${{ github.ref_name == 'main' }}    # returns true or false
${{ github.sha }}                    # returns the commit SHA value
${{ secrets.MY_SECRET }}            # returns the secret value (masked)
```

Expressions can be used:
```
→ In run: steps         → echo "${{ github.actor }}"
→ In if: conditions     → if: github.ref_name == 'main'
→ In with: inputs       → java-version: ${{ env.JAVA_VERSION }}
→ In name: fields       → name: Deploy to ${{ inputs.environment }}
→ In env: values        → IMAGE_TAG: ${{ github.sha }}
```

---

## 2. Operators

**09.1.1-expression-syntax-operators.yml**

```yaml
name: 09.1.1-Expression-Syntax-Operators

on:
  workflow_dispatch:
    inputs:
      environment:
        type: choice
        options: [ development, staging, production ]
        default: development

jobs:
  operators-demo:
    runs-on: ubuntu-latest
    steps:
      - name: Comparison operators
        run: |
          echo "ref_name == main : ${{ github.ref_name == 'main' }}"
          echo "ref_name != main : ${{ github.ref_name != 'main' }}"
          echo "run_number > 0   : ${{ github.run_number > 0 }}"

      - name: Logical operators
        run: |
          echo "push AND main    : ${{ github.event_name == 'push' && github.ref_name == 'main' }}"
          echo "push OR dispatch : ${{ github.event_name == 'push' || github.event_name == 'workflow_dispatch' }}"
          echo "NOT push         : ${{ !(github.event_name == 'push') }}"

      - name: Only production
        if: inputs.environment == 'production'
        run: echo "Production step runs ✅"

      - name: Not production
        if: inputs.environment != 'production'
        run: echo "Non-production step runs ✅"

      - name: Manual staging
        if: github.event_name == 'workflow_dispatch' && inputs.environment == 'staging'
        run: echo "Manual staging deployment ✅"

      - name: Lower environments
        if: inputs.environment == 'development' || inputs.environment == 'staging'
        run: echo "Lower environment step runs ✅"
```

**Comparison operators:**

| Operator | Meaning | Example |
|---|---|---|
| `==` | Equal | `github.ref_name == 'main'` |
| `!=` | Not equal | `github.ref_name != 'main'` |
| `>` | Greater than | `github.run_number > 0` |
| `<` | Less than | `github.run_number < 100` |
| `>=` | Greater or equal | `github.run_number >= 1` |
| `<=` | Less or equal | `github.run_number <= 100` |

**Logical operators:**

| Operator | Meaning | Example |
|---|---|---|
| `&&` | AND — both must be true | `event == 'push' && ref == 'main'` |
| `\|\|` | OR — at least one must be true | `event == 'push' \|\| event == 'dispatch'` |
| `!` | NOT — inverts the result | `!(github.event_name == 'push')` |

---

## 3. String Functions

**09.2.1-string-functions.yml**

```yaml
name: 09.2.1-String-Functions

on:
  workflow_dispatch:

jobs:
  string-functions-demo:
    runs-on: ubuntu-latest
    steps:
      - name: contains()
        run: |
          echo "ref contains main        : ${{ contains(github.ref, 'main') }}"
          echo "repo contains i27academy  : ${{ contains(github.repository, 'i27academy') }}"

      - name: Use contains() in condition
        if: contains(github.repository, 'i27academy')
        run: echo "Repository belongs to i27academy ✅"

      - name: startsWith()
        run: |
          echo "ref starts with refs/heads : ${{ startsWith(github.ref, 'refs/heads') }}"
          echo "ref starts with refs/tags  : ${{ startsWith(github.ref, 'refs/tags') }}"

      - name: endsWith()
        run: |
          echo "repo ends with dashboard   : ${{ endsWith(github.repository, 'dashboard') }}"
          echo "repo ends with private     : ${{ endsWith(github.repository, 'private') }}"

      - name: format()
        run: |
          echo "${{ format('Actor: {0} | Branch: {1} | Run: {2}', github.actor, github.ref_name, github.run_number) }}"
          echo "${{ format('Image: devopswithcloudhub/i27academy:{0}', github.sha) }}"
```

**String functions reference:**

| Function | What it does | Example |
|---|---|---|
| `contains(str, val)` | Check if string contains value | `contains(github.ref, 'main')` |
| `startsWith(str, val)` | Check if string starts with value | `startsWith(github.ref, 'refs/tags')` |
| `endsWith(str, val)` | Check if string ends with value | `endsWith(github.repository, 'private')` |
| `format(str, {0}, {1})` | Build a formatted string | `format('{0}:{1}', vars.IMAGE, github.sha)` |
| `join(array, separator)` | Join array values into string | `join(github.event.commits.*.message, ', ')` |

---

## 4. Data Functions

**09.3.1-data-functions.yml**

```yaml
name: 09.3.1-Data-Functions

on:
  workflow_dispatch:

jobs:
  data-functions-demo:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: toJson()
        env:
          RUNNER_CONTEXT: ${{ toJson(runner) }}
        run: |
          echo "Runner context:"
          echo "$RUNNER_CONTEXT"

      - name: fromJson()
        run: |
          echo "Valid env check: ${{ contains(fromJson('["development","staging","production"]'), 'development') }}"

      - name: hashFiles()
        run: |
          echo "pom.xml hash  : ${{ hashFiles('pom.xml') }}"
          echo "all yaml hash : ${{ hashFiles('**/*.yml') }}"
```

**Data functions reference:**

| Function | What it does | Common use |
|---|---|---|
| `toJson(value)` | Convert object to JSON string | Debugging contexts |
| `fromJson(string)` | Parse JSON string into object | Dynamic matrix, membership check |
| `hashFiles(path)` | Hash file contents | Cache keys |

**toJson() important rule:**

```yaml
# ✅ Always assign to env var first
- name: Dump context
  env:
    CTX: ${{ toJson(github) }}
  run: echo "$CTX"

# ❌ Never put directly in run:
- name: Dump context
  run: echo "${{ toJson(github) }}"   # can cause YAML issues
```

**fromJson() for membership check:**

```yaml
if: contains(fromJson('["development","staging","production"]'), inputs.environment)
```

**hashFiles() for cache keys — covered in detail in Module 10.**

---

## 5. Status Check Functions

Status check functions control whether a step or job runs based on what happened before it.

**09.4.1-status-check-functions.yml**

```yaml
name: 09.4.1-Status-Check-Functions

on:
  workflow_dispatch:

jobs:
  status-functions-demo:
    runs-on: ubuntu-latest
    steps:
      - name: Step 1 — runs normally
        run: echo "Step 1 completed"

      - name: Step 2 — fails intentionally
        run: exit 1
        continue-on-error: true

      - name: Step 3 — success() — skipped
        if: success()
        run: echo "Only runs if all previous steps passed ✅"

      - name: Step 4 — failure() — runs
        if: failure()
        run: echo "A previous step failed — running alert ⚠️"

      - name: Step 5 — always() — always runs
        if: always()
        run: echo "Always runs — perfect for reports ✅"

  notify-on-failure:
    runs-on: ubuntu-latest
    needs: status-functions-demo
    if: failure()
    steps:
      - name: Send alert
        run: echo "Job failed — sending notification"

  final-summary:
    runs-on: ubuntu-latest
    needs: [ status-functions-demo, notify-on-failure ]
    if: always()
    steps:
      - name: Write summary
        run: |
          echo "## Pipeline Summary" >> $GITHUB_STEP_SUMMARY
          echo "| Job | Result |" >> $GITHUB_STEP_SUMMARY
          echo "| --- | --- |" >> $GITHUB_STEP_SUMMARY
          echo "| status-functions-demo | ${{ needs.status-functions-demo.result }} |" >> $GITHUB_STEP_SUMMARY
          echo "| notify-on-failure | ${{ needs.notify-on-failure.result }} |" >> $GITHUB_STEP_SUMMARY
```

Observe:
```
→ Step 2 fails (exit 1) but continue-on-error: true lets workflow continue
→ Step 3 (success()) is skipped — a step already failed
→ Step 4 (failure()) runs — a step failed
→ Step 5 (always()) runs — regardless of what happened
→ notify-on-failure job runs — because status-functions-demo had failures
→ final-summary job runs — because if: always()
```

**Status check functions reference:**

| Function | When it runs | Use for |
|---|---|---|
| `success()` | All previous steps/jobs passed | Default — no need to write it explicitly |
| `failure()` | A previous step/job failed | Alerts, cleanup on failure |
| `always()` | No matter what | Upload reports, write summary, cleanup |
| `cancelled()` | Workflow was cancelled | Notify team of cancellation |

**Jenkins equivalent:**

```groovy
// Jenkins
post {
  always  { }    →    if: always()
  failure { }    →    if: failure()
  success { }    →    if: success()
}
```

---

## 6. if: Conditions — Real World Patterns

**09.5.1-if-conditions-real-world.yml**

```yaml
name: 09.5.1-If-Conditions-Real-World

on:
  push:
    branches: [ main, develop, 'release/**' ]
  workflow_dispatch:
    inputs:
      environment:
        type: choice
        options: [ development, staging, production ]
        default: development

jobs:
  real-world-conditions:
    runs-on: ubuntu-latest
    steps:
      - name: Only on main branch
        if: github.ref_name == 'main'
        run: echo "Running on main branch ✅"

      - name: Only on push to main
        if: github.event_name == 'push' && github.ref_name == 'main'
        run: echo "Push to main ✅"

      - name: Only on version tags
        if: startsWith(github.ref, 'refs/tags/v')
        run: echo "Version tag detected ✅"

      - name: Only on release branches
        if: startsWith(github.ref_name, 'release/')
        run: echo "Release branch ✅"

      - name: Skip when commit message has skip-ci
        if: "!contains(github.event.head_commit.message, '[skip-ci]')"
        run: echo "Running — no skip-ci in commit message ✅"

      - name: Only for valid environments
        if: contains(fromJson('["development","staging","production"]'), inputs.environment)
        run: echo "Valid environment: ${{ inputs.environment }} ✅"
```

**Patterns you will use constantly:**

```yaml
# Only on main
if: github.ref_name == 'main'

# Only on version tags (v1.0.0, v2.3.1)
if: startsWith(github.ref, 'refs/tags/v')

# Only on release branches
if: startsWith(github.ref_name, 'release/')

# Skip when commit message says so
if: "!contains(github.event.head_commit.message, '[skip-ci]')"

# Validate environment input
if: contains(fromJson('["development","staging","production"]'), inputs.environment)

# Only on manual trigger to production
if: inputs.environment == 'production' && github.event_name == 'workflow_dispatch'
```

---

## Module 09 Summary

- Expression syntax — `${{ }}` — GitHub evaluates what is inside
- Comparison operators — `==`, `!=`, `>`, `<`, `>=`, `<=`
- Logical operators — `&&` (AND), `||` (OR), `!` (NOT)
- `contains()` — check if a string contains a value
- `startsWith()` — check if a string starts with a value
- `endsWith()` — check if a string ends with a value
- `format()` — build a formatted string with placeholders
- `join()` — join array values into a string
- `toJson()` — convert context to JSON — always assign to env var first
- `fromJson()` — parse JSON string — useful for membership checks
- `hashFiles()` — hash file contents — used for cache keys (Module 10)
- `success()` — default — runs only if all previous steps/jobs passed
- `failure()` — runs only if a previous step/job failed
- `always()` — runs no matter what — use for reports, cleanup, summary
- `cancelled()` — runs only if workflow was cancelled

---

## File Summary

| File | What it demonstrates |
|---|---|
| 09.1.1-expression-syntax-operators.yml | ==, !=, &&, \|\|, ! operators in conditions |
| 09.2.1-string-functions.yml | contains, startsWith, endsWith, format, join |
| 09.3.1-data-functions.yml | toJson, fromJson, hashFiles |
| 09.4.1-status-check-functions.yml | success, failure, always, cancelled |
| 09.5.1-if-conditions-real-world.yml | Real world patterns for if: conditions |

---

*i27Academy · GitHub Actions Course · Module 09 · i27academy.com*
