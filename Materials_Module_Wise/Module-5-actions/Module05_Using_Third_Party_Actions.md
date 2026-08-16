# Module 05 — Using Third-Party Actions
### i27Academy — GitHub Actions Course

---

## Agenda

1. The problem without actions
2. The solution with actions
3. $GITHUB_ENV and $GITHUB_PATH
4. What is an action
5. Finding actions in the marketplace
6. Version pinning
7. Evaluating third-party actions

---

## 1. The Problem Without Actions

Every CI/CD pipeline needs to do basic things — clone the repository, set up a language runtime, install dependencies. Without actions, you write all of this yourself using shell commands.

**05.1.1-without-actions.yml**

```yaml
name: 05.1.1-Without-Actions

on:
  workflow_dispatch:

jobs:
  manual-setup:
    runs-on: ubuntu-latest
    steps:
      - name: Manually clone the repository
        run: |
          git clone https://github.com/${{ github.repository }}.git .
          git checkout ${{ github.sha }}
          echo "Repository cloned successfully"
          ls -la

      - name: Manually install Java 21
        run: |
          echo "Downloading Java 21..."
          wget -q https://download.java.net/java/GA/jdk21.0.2/f2283984656d49d69e91c558476027ac/13/GPL/openjdk-21.0.2_linux-x64_bin.tar.gz
          echo "Extracting Java..."
          tar -xzf openjdk-21.0.2_linux-x64_bin.tar.gz
          echo "Setting up JAVA_HOME..."
          export JAVA_HOME=$PWD/jdk-21.0.2
          export PATH=$JAVA_HOME/bin:$PATH
          echo "JAVA_HOME=$PWD/jdk-21.0.2" >> $GITHUB_ENV
          echo "$PWD/jdk-21.0.2/bin" >> $GITHUB_PATH
          echo "Java installed successfully"

      - name: Verify Java installation
        run: |
          java -version
          echo "JAVA_HOME: $JAVA_HOME"

      - name: Build the project
        run: mvn package -DskipTests
```

Look at how much work is needed just to:
- Clone the repository
- Install Java 21

And this still does not handle:
```
→ Java version conflicts with other tools on the runner
→ Maven dependency cache — downloads everything fresh every run
→ Download failures — no retry logic
→ Cross-platform support — works on Linux only
→ JAVA_HOME not set correctly in all cases
```

Every team that uses this approach ends up maintaining these shell scripts themselves.

---

## 2. The Solution With Actions

**05.1.2-with-actions.yml**

```yaml
name: 05.1.2-With-Actions

on:
  workflow_dispatch:

jobs:
  with-actions:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Setup Java 21
        uses: actions/setup-java@v4
        with:
          java-version: '21'
          distribution: temurin
          cache: maven

      - name: Verify Java installation
        run: |
          java -version
          echo "JAVA_HOME: $JAVA_HOME"

      - name: Build the project
        run: mvn package -DskipTests
```

Exact same result — but:

```
actions/checkout@v4
  → Replaces 5 lines of git commands
  → Handles authentication automatically
  → Works on all operating systems

actions/setup-java@v4
  → Replaces 10 lines of manual installation
  → Pins the exact Java version
  → Sets JAVA_HOME correctly
  → Caches Maven dependencies automatically
  → Works on Linux, Windows and macOS
  → Maintained by GitHub — always up to date
```

**Side by side:**

```
Without actions              With actions
──────────────────────────   ──────────────────────────
20+ lines of shell script    2 steps with uses:
Manual git clone             actions/checkout@v4
Manual Java download         actions/setup-java@v4
No caching                   cache: maven built in
Linux only                   All OS supported
You maintain the scripts     GitHub maintains the action
```

---

---

## 3. $GITHUB_ENV and $GITHUB_PATH

In `05.1.1-without-actions.yml` we used two special files — `$GITHUB_ENV` and `$GITHUB_PATH`. Let us understand what they are and why they are needed.

**Why they exist:**

Each step runs in its own shell process. When a step finishes its process ends — any `export` or `PATH=` changes made inside that step are lost.

```yaml
steps:
  - name: Set a variable
    run: export MY_VAR=hello     # lives only in this step's process

  - name: Use the variable
    run: echo $MY_VAR            # prints nothing — process is gone
```

`$GITHUB_ENV` and `$GITHUB_PATH` are GitHub's solution — files on the runner that GitHub reads between steps and injects into the next step's environment.

---

**`$GITHUB_ENV` — set environment variables across steps:**

```yaml
steps:
  - name: Set a variable
    run: echo "MY_VAR=hello" >> $GITHUB_ENV

  - name: Use the variable
    run: echo "Value is $MY_VAR"    # prints: hello
```

We used it in `05.1.1` to set `JAVA_HOME`:

```yaml
echo "JAVA_HOME=$PWD/jdk-21.0.2" >> $GITHUB_ENV
```

---

**`$GITHUB_PATH` — add directories to PATH across steps:**

```yaml
steps:
  - name: Add Java bin to PATH
    run: echo "$PWD/jdk-21.0.2/bin" >> $GITHUB_PATH

  - name: Use Java
    run: java -version    # works — bin folder is now in PATH
```

We used it in `05.1.1` to make the Java executable available:

```yaml
echo "$PWD/jdk-21.0.2/bin" >> $GITHUB_PATH
```

---

**Key difference:**

```
$GITHUB_ENV    → sets a variable (KEY=VALUE)
               → echo "MY_VAR=hello" >> $GITHUB_ENV
               → access as $MY_VAR in later steps

$GITHUB_PATH   → adds a folder to PATH
               → echo "/path/to/bin" >> $GITHUB_PATH
               → executables in that folder are findable in later steps
```

**How it works:**

```
Step 1 runs → writes to $GITHUB_ENV file on disk
Step 1 ends → shell process exits
GitHub reads the file
Step 2 starts → GitHub injects the values before Step 2 begins
Step 2 can now access MY_VAR
```

Key points:
```
→ $GITHUB_ENV  — variables persist across steps in the same job
→ $GITHUB_PATH — PATH additions persist across steps in the same job
→ Neither works across jobs — jobs run on separate machines
→ For cross-job data use job outputs (covered in Module 07)
→ actions/setup-java handles all of this automatically — no manual work needed
```

---

## 4. What is an Action

An action is a pre-built, reusable unit of code that performs a specific task. It is packaged as a GitHub repository with an `action.yml` file that defines what it does and what inputs it accepts.

```
uses: actions/setup-java@v4
       └──────┬──────┘  └┬┘
          owner/repo    version
```

This line tells GitHub Actions:
- Go to `github.com/actions/setup-java`
- Get the code at version `v4`
- Run it as a step

**Three types of actions:**

```
Composite    → a collection of steps in a YAML file
               most practical — no coding needed
               we build these in Module 13

JavaScript   → Node.js code that runs on the runner
               used for complex logic and API calls

Docker       → runs inside a container
               used for specific environment requirements
```

For this course and for real-world DevOps use — **composite actions** are what you will use and build.

**Passing inputs to an action using `with:`:**

```yaml
- uses: actions/setup-java@v4
  with:                         # inputs the action accepts
    java-version: '21'          # which Java version to install
    distribution: temurin       # which Java distribution
    cache: maven                # enable Maven caching
```

Every action defines its own inputs in its `action.yml`. The `with:` block is how you pass values to those inputs.

---

## 4. Finding Actions in the Marketplace

The GitHub Marketplace has 10,000+ pre-built actions for almost everything you need.

```
github.com/marketplace?type=actions
```

You can search by category:
```
CI             → build, test, lint
Deployment     → AWS, GCP, Azure, Kubernetes
Docker         → build, push, scan
Security       → code scanning, secret scanning
Notifications  → Slack, Teams, email
Utilities      → cache, artifact, upload
```

**Most commonly used actions:**

| Action | Purpose |
|---|---|
| `actions/checkout@v4` | Clone repository onto the runner |
| `actions/setup-java@v4` | Install Java — any version and distribution |
| `actions/setup-node@v4` | Install Node.js |
| `actions/setup-python@v5` | Install Python |
| `actions/upload-artifact@v4` | Save files after a job |
| `actions/download-artifact@v4` | Retrieve saved files in another job |
| `actions/cache@v4` | Cache dependencies between runs |
| `docker/login-action@v3` | Login to Docker Hub |
| `docker/build-push-action@v6` | Build and push Docker images |

**Official actions — owned by GitHub or the tool vendor:**

```
actions/*           → maintained by GitHub
docker/*            → maintained by Docker
aws-actions/*       → maintained by AWS
google-github-actions/* → maintained by Google
azure/*             → maintained by Microsoft
```

These are the safest to use — the tool vendor maintains them.

---

## 5. Version Pinning

Every action reference includes a version:

```yaml
uses: actions/checkout@v4
#                      ↑
#                   version tag
```

**Three ways to reference a version — in order of safety:**

**1. Branch reference — never use in production:**
```yaml
uses: actions/checkout@main    # whatever is on main right now
                               # changes without warning ❌
```

**2. Version tag — safe for official actions:**
```yaml
uses: actions/checkout@v4      # major version tag
                               # gets patches automatically ✅
```

**3. Exact SHA — safest for third-party actions:**
```yaml
uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683  # v4.2.2
                               # this exact code, forever ✅✅
```

**Rule of thumb:**

```
Official actions (actions/*, docker/*, aws-actions/*)
  → @v4 tag is fine — low risk

Third-party actions (unknown org or individual)
  → Always pin to exact SHA
  → Add the version as a comment so humans can read it

Anything touching secrets or cloud credentials
  → Always pin to SHA — no exceptions
```

**How to find the SHA for an action:**

```
1. Go to the action repo on GitHub
2. Click Releases
3. Find the version you want
4. Copy the full commit SHA
```

---

## 6. Evaluating Third-Party Actions

Before using any action that is not from an official publisher, run through this checklist:

```
✅ Who publishes it?
   → Verified creator badge or known org (docker/*, aws-actions/*)
   → Unknown individual with no other public work? Be cautious.

✅ How many stars and how widely used?
   → Thousands of stars and used by major projects = safer
   → A handful of stars created last month = be careful

✅ When was it last updated?
   → Active maintenance and recent commits = good
   → Last commit 2 years ago, open issues unanswered = risk

✅ Does the source code make sense?
   → Composite actions: read the action.yml — every step is visible
   → JavaScript actions: is the source readable or obfuscated?

✅ Does it ask for more permissions than it needs?
   → An action that builds Docker images should not need write access to issues

✅ Any known security issues?
   → Search: "[action name] CVE" or check GitHub Security Advisories
```

**A simple rule:**

```
If an action is doing something simple — like sending a Slack message
or running a shell command — consider writing it yourself as a
composite action instead of depending on a third party.

If an action is complex and from a trusted publisher — use it
and pin it to a SHA.
```

---

## Module 05 Summary

- Without actions — you write and maintain complex shell scripts yourself
- Actions replace those scripts with a single `uses:` line
- An action is a pre-built reusable task packaged as a GitHub repository
- Three types: composite (YAML), JavaScript (Node.js), Docker
- Use `with:` to pass inputs to an action
- Find actions at github.com/marketplace?type=actions
- Always pin to a version tag — never use `@main` or `@latest`
- Pin third-party or credential-touching actions to exact SHA
- Evaluate third-party actions before using — publisher, stars, activity, source

---



---

*i27Academy · GitHub Actions Course · Module 05 · i27academy.com*
