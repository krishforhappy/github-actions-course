### What is github actions ? 
* Whats the first step to start implementing github actions 
    * need to create workflows  ? 
        * .github/workflows/workflow-name.yaml 
* workflow > automation file  (.yaml or .yml)
* Major blocks in workflows 
    * workflow > automation file  (.yaml or .yml)
    * Event > what triggers the workflow 
    * jobs >  job > a group of steps on one machine(runner)
    * steps > single task inside a job 
    * action > pre-built reusable task 
* Jobs: 
    * Muliptle jobs in a workflow 
    * mulitple jobs runn in parealll by default 
    * if i want to have dependncey on other jobs , needs: 
    * each job starts in a fresh vm ===> no shared state 
* steps;
    * steps runs SEQUENTIALLY -> one after other 
    * actually we will be implementing tasks 
        * in run i can have task been done in 2 ways:
            * run: shell commands 
            * uses: using action 
    * step fails:
        * remaining steps in the same job are SKIPPED 
        * Job is marked as FAILURE 
        * Jobs with needs: pointing to the failed jobs are SKIPEED 
        * Jobs with no needs: running 
* Job Status:
    * success 
    * failure 
    * cancelled 
    * skipped 

## Events:
* Events is an acitvity that happens in and around your repository, that trigger a worflow 
* represented using `on:`
* event types:
    * push 
        * branches: [ main, feature]
        * paths: 
            - src/*
        * branches-ignore: 
        * paths-ignore: 
    * pull_request
        * opened 
        * reopened 
        * synchronize 
    * schedule 
    * workflow_dispatch
        * adding a run workflow button in the github UI 
    * workflow_run
        * one workflow after other 
        * types: completed
## Runners: 
* runs-on: 
    * ubuntu-latest
        * cost multiplier : 1x
    * windows-latest
        * cost multiplier : 2x
    * macos-latest
        * cost multiplier : 10x 
* pre installed tools:
    * java , nodejs, docker, kubeectl, git maven, gcloud, aws, az
* runners.context 
    * runners.os
    * runners.name 
* GITHUB_ENV 
    * set env varaibles that persists across the steps in the same job
* GITHUB_PATH
    * add directories to the path that perisst across job

## actions:
* we havent wrote any custom actions, we shall do them in a diff module. 
* Actions is a prebuilt reusable code that perform a specific task. 
* uses: actions/setup-java@v1
* actions/setup-java =========> owner/repo
* v1=============> version 
* @branch ========> Never, changes without warning and not recommended
* @tagname =======> Safer for official actions 
* @SHA ===========> safest ==
* three types of actions:
    * composite 
        * yaml steps ====> most practical for devops 
    * javascript
        * 
    * docker
        * but  
* most common actions
```
actions/checkout@v4
actions/setup-java@v4
actions/setup-node@v4
actions/upload-artifact@v4
docker/login-action@v1
docker/build-push-action@v1
google-github-actions/setup-gcloud@v1
```
## Environmental variable and secrets 
* env is different Environment(we havent discussed) is differnet in github actions 
* env: static values 
* env: three level scopes
    * workflow level
    * job level 
    * step level 
* precendence rule:
    * step level > job level > workflow level
* dunamic values 
    * $GITHUB_ENV 
    * echo "x=10"  >> $GITHUB_ENV
    * The x value will be available from next step onwards, not in the same step .
    * and x values is scoped to same job only, and does not cross multiple jobs. 
    * for cross job data we use job outputs
* vars.**
    * ${{ vars.variable-name }}
    * github > org 
    * github >repo 
    * except senstivie data
* secrets.** 
    * ${{ secrets.secret-name}}
* stored in githb ui ===. not in the yaml 
* variables and secrets are defined at
    * Repo
    * org
    * Environment (we havent seen yet)

*     SONAR_URL = https://sonar.hsbc.com

## outputs
* if i want to have data shared across jobs 
* outputs can be called in the same job across steps, or same workflow across jobs

* using `id:` in same job
    * run: echo ${{ steps.is.outputs.key }}
* across job:
```
# definition
jobs: 
  build-job:
    outputs:
      my-id-output: ${{ steps.my-id.outputs.x}}
steps:
- name: step-name
  id: my-id
  run: echo "x=10" >> $GITHUB_OUTPUT


# call from other job 
deploy:
  needs: build-job
  steps:
    run: ${{ needs.build-job.outputs.my-id-output.x}}
```

## Context:
* ${{ context.property }}
* github.*** # repo, event, ....
* runner.*** # machine infor
* env.***       # Workflow env variables
* vars.***      # Github UI variables
* secret.***    # Github UI Secrets 
* steps.****    # presivos step outputs > same job
* needs.****    # previous job outputs 
* inputs.****   # workflow_disapatch or workflow_call 
* matrix.**** [ we havent seent his ]


## inputs 
* type
    * choice
    * string 
    * number 
    * boolean

* Expressions , Functions 
* Artifact , caching 