# Module 11 — Working with Matrices
### i27Academy — GitHub Actions Course

---

## Agenda

1. What is a matrix strategy
2. Single dimension matrix
3. Multi-dimensional matrix
4. include and exclude
5. fail-fast and max-parallel
6. Dynamic matrix
7. matrix.* context

---

## 1. What is a Matrix Strategy

A matrix strategy lets you run the same job multiple times with different inputs — automatically and in parallel.

Without matrix — repetitive and hard to maintain:
```yaml
jobs:
  test-java-17:
    steps:
      - uses: actions/setup-java@v4
        with: { java-version: '17' }
      - run: mvn test

  test-java-21:
    steps:
      - uses: actions/setup-java@v4
        with: { java-version: '21' }
      - run: mvn test
```

With matrix — define once, runs for every value:
```yaml
jobs:
  test:
    strategy:
      matrix:
        java-version: [ '17', '21' ]
    steps:
      - uses: actions/setup-java@v4
        with:
          java-version: ${{ matrix.java-version }}
      - run: mvn test
```

Same result — but defined once. GitHub creates one job per matrix value and runs them in parallel.

---

## 2. Single Dimension Matrix

**11.1.1-matrix-basic.yml**

```yaml
name: 11.1.1-Matrix-Basic

on:
  workflow_dispatch:

jobs:
  test:
    name: Test - Java ${{ matrix.java-version }}
    runs-on: ubuntu-latest
    strategy:
      matrix:
        java-version: [ '17', '21' ]

    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-java@v4
        with:
          java-version: ${{ matrix.java-version }}
          distribution: temurin
          cache: maven

      - run: mvn test

      - name: Print matrix value
        run: |
          echo "Running on Java ${{ matrix.java-version }}"
          echo "Runner OS   : ${{ runner.os }}"
```

Observe:
```
→ Two parallel jobs created:
    Test - Java 17
    Test - Java 21
→ Both start at the same time
→ Each runs on its own fresh Ubuntu VM
→ Both must pass for the workflow to succeed
→ matrix.java-version holds the current value in each job
```

---

## 3. Multi-Dimensional Matrix

Two or more dimensions — GitHub creates all combinations.

**11.1.2-matrix-multi-dimension.yml**

```yaml
name: 11.1.2-Matrix-Multi-Dimension

on:
  workflow_dispatch:

jobs:
  test:
    name: Test - Java ${{ matrix.java-version }} on ${{ matrix.os }}
    runs-on: ${{ matrix.os }}
    strategy:
      matrix:
        java-version: [ '17', '21' ]
        os: [ ubuntu-latest, windows-latest ]

    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with:
          java-version: ${{ matrix.java-version }}
          distribution: temurin
          cache: maven
      - run: mvn test
      - run: |
          echo "Java version : ${{ matrix.java-version }}"
          echo "OS           : ${{ matrix.os }}"
```

Observe:
```
→ 4 parallel jobs created (2 java × 2 OS):
    Test - Java 17 on ubuntu-latest
    Test - Java 17 on windows-latest
    Test - Java 21 on ubuntu-latest
    Test - Java 21 on windows-latest
→ runs-on: ${{ matrix.os }} — runner changes per job
```

**Combinations created:**

| Job | Java | OS |
|---|---|---|
| 1 | 17 | ubuntu-latest |
| 2 | 17 | windows-latest |
| 3 | 21 | ubuntu-latest |
| 4 | 21 | windows-latest |

---

## 4. include and exclude

**11.2.1-matrix-include-exclude.yml**

```yaml
name: 11.2.1-Matrix-Include-Exclude

on:
  workflow_dispatch:

jobs:
  test:
    name: Test - Java ${{ matrix.java-version }} on ${{ matrix.os }}
    runs-on: ${{ matrix.os }}
    strategy:
      matrix:
        java-version: [ '17', '21' ]
        os: [ ubuntu-latest, windows-latest ]

        exclude:
          - java-version: '17'
            os: windows-latest

        include:
          - java-version: '21'
            os: ubuntu-latest
            is-primary: true

    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with:
          java-version: ${{ matrix.java-version }}
          distribution: temurin
          cache: maven
      - run: mvn test

      - name: Primary combination marker
        if: matrix.is-primary == true
        run: echo "This is the PRIMARY test combination ✅"
```

**exclude — remove specific combinations:**
```yaml
exclude:
  - java-version: '17'
    os: windows-latest    # removes this combination
```

After exclude — 3 jobs run (not 4):
```
Test - Java 17 on ubuntu  ✅ kept
Test - Java 21 on ubuntu  ✅ kept (is-primary = true)
Test - Java 21 on windows ✅ kept
Test - Java 17 on windows ❌ excluded
```

**include — add extra variables to existing combinations:**
```yaml
include:
  - java-version: '21'
    os: ubuntu-latest
    is-primary: true     # extra variable added to this combination
```

`include` does NOT add a new job — it adds extra variables to an existing combination. Those variables are then accessible via `matrix.is-primary`.

---

## 5. fail-fast and max-parallel

**11.2.2-matrix-fail-fast-max-parallel.yml**

```yaml
strategy:
  fail-fast: false
  max-parallel: 2
  matrix:
    java-version: [ '17', '21' ]
```

**fail-fast:**

| Setting | Behaviour | Use when |
|---|---|---|
| `true` (default) | One job fails → all others cancelled immediately | Fast feedback — stop everything on failure |
| `false` | One job fails → others continue running | See ALL results — which versions pass/fail |

**max-parallel:**
```yaml
max-parallel: 2    # run at most 2 matrix jobs at a time
```

Default is all matrix jobs run at the same time. Use `max-parallel` to limit concurrency when runners are limited or resource usage needs controlling.

---

## 6. Dynamic Matrix

Generate the matrix at runtime instead of hardcoding values in the YAML file.

**11.3.1-matrix-dynamic.yml**

```yaml
name: 11.3.1-Matrix-Dynamic

on:
  workflow_dispatch:

jobs:
  prepare-matrix:
    name: Prepare Matrix
    runs-on: ubuntu-latest
    outputs:
      matrix: ${{ steps.set-matrix.outputs.matrix }}
    steps:
      - name: Set dynamic matrix
        id: set-matrix
        run: |
          echo 'matrix={"java-version":["17","21"],"include":[{"java-version":"21","is-lts":true},{"java-version":"17","is-lts":false}]}' >> $GITHUB_OUTPUT

  test:
    name: Test - Java ${{ matrix.java-version }}
    runs-on: ubuntu-latest
    needs: prepare-matrix
    strategy:
      matrix: ${{ fromJson(needs.prepare-matrix.outputs.matrix) }}

    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with:
          java-version: ${{ matrix.java-version }}
          distribution: temurin
          cache: maven
      - run: mvn test
      - name: LTS marker
        if: matrix.is-lts == true
        run: echo "Java ${{ matrix.java-version }} is LTS ✅"
```

Observe:
```
→ prepare-matrix job generates a JSON matrix string
→ test job uses fromJson() to parse it into a matrix
→ Two jobs created: Test - Java 17 and Test - Java 21
→ is-lts variable added per Java version via include
```

**Why dynamic matrix:**
```
Static matrix  → values hardcoded in YAML
               → change versions = edit the file

Dynamic matrix → values computed at runtime
               → can come from: API, config file, logic
               → change versions = update source, not workflow
```

---

## 7. matrix.* Context

Inside a matrix job, `matrix.*` gives you access to the current combination's values.

```yaml
strategy:
  matrix:
    java-version: [ '17', '21' ]
    os: [ ubuntu-latest, windows-latest ]

steps:
  - run: |
      echo "Java    : ${{ matrix.java-version }}"
      echo "OS      : ${{ matrix.os }}"
      echo "Primary : ${{ matrix.is-primary }}"
```

Common uses:
```yaml
# Use in job name
name: Test - Java ${{ matrix.java-version }} on ${{ matrix.os }}

# Use in runs-on
runs-on: ${{ matrix.os }}

# Use in action inputs
java-version: ${{ matrix.java-version }}

# Use in conditions
if: matrix.is-primary == true

# Use in artifact names (avoid duplicates)
name: surefire-java${{ matrix.java-version }}-${{ matrix.os }}
```

**Artifact naming with matrix — important:**
```yaml
- uses: actions/upload-artifact@v4
  with:
    name: surefire-java${{ matrix.java-version }}
    # Each matrix job uploads with a unique name
    # Without this — jobs overwrite each other's artifacts
```

---

## Module 11 Summary

- Matrix strategy runs the same job multiple times with different inputs — in parallel
- `matrix:` defines the dimensions and values
- `${{ matrix.KEY }}` accesses the current value in each job
- Multi-dimensional matrix — all combinations are created automatically
- `exclude:` removes specific combinations from the matrix
- `include:` adds extra variables to existing combinations — does not add new jobs
- `fail-fast: false` — continue all jobs even if one fails — see all results
- `fail-fast: true` (default) — cancel all on first failure — fast feedback
- `max-parallel:` limits how many matrix jobs run concurrently
- Dynamic matrix — generate matrix at runtime using `fromJson()`
- Always use unique artifact names in matrix jobs — include `matrix.KEY` in the name

---

## File Summary

| File | What it demonstrates |
|---|---|
| 11.1.1-matrix-basic.yml | Single dimension — 2 Java versions |
| 11.1.2-matrix-multi-dimension.yml | 2 dimensions — Java × OS = 4 jobs |
| 11.2.1-matrix-include-exclude.yml | exclude combination + include extra variable |
| 11.2.2-matrix-fail-fast-max-parallel.yml | fail-fast: false + max-parallel |
| 11.3.1-matrix-dynamic.yml | fromJson() dynamic matrix |

---

*i27Academy · GitHub Actions Course · Module 11 · i27academy.com*
