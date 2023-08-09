---
theme: default
class: text-center
highlighter: shiki
lineNumbers: false
drawings:
  persist: false
title: A comedy of Airflows
---

# A comedy of Airflows

Running dbt core in production the least bad way

Dean Verhey

Seattle dbt Meetup August 2023

---
layout: default
---

# Who am I?

- 💼 **Currently** - Senior Data/Analytics Engineer at [LaunchDarkly](https://launchdarkly.com/)
- 👋 **Previously** - Data and/or software engineering at:
    - 💵 [Coupa Software](https://www.coupa.com/)
    - ✈️ [Yapta](https://www.geekwire.com/2020/coupa-software-acquiring-seattle-startup-yapta-help-businesses-cut-travel-costs/) (acquired by Coupa in 2020)
    - 🚒 [Emergency Reporting](https://emergencyreporting.com/)
    - 🎓 Western Washington University
    - 🛫 University of North Dakota
- 🏔️ **When not working** - Skiing, spending time with my partner and [our cat](https://deancat.netlify.app/)

TODO: Headshot or other info

<style>
h1 {
  background-color: #2B90B6;
  background-image: linear-gradient(45deg, #4EC5D4 10%, #146b8c 20%);
  background-size: 100%;
  -webkit-background-clip: text;
  -moz-background-clip: text;
  -webkit-text-fill-color: transparent;
  -moz-text-fill-color: transparent;
}
</style>

---
layout: default
---

# Agenda

<Toc></Toc>

---
layout: statement
level: 2
---

<style>
h1, h3 {color: #2B90B6;}
</style>

# _Goals_

## Know what you're getting yourself into

<br><br>

### _Help me help you:_

How many here use dbt?

How many don't and are considering implementation?

How many are responsible for running dbt in production?

---
layout: two-cols
---

# Production dbt-ing

- Can be as simple as the CLI operations you’re already familiar with
- Not very resource-intensive
  - Meaningful compute handled by DWH
- Does write out a lot of files
  - `target-path`, `packages-install-path`, `log-path` can help with this
- Still requires a Python env
- New in 1.5 - [programmatic invocations](https://docs.getdbt.com/reference/programmatic-invocations)

::right::

```bash
❯ dbt run
04:05  Running with dbt=1.5.2
04:05  Registered adapter: duckdb=1.5.2
04:05  ...
04:05  ...
04:05  ...
04:05  Completed successfully
04:05
04:05  Done. PASS=7 WARN=0 ERROR=0 SKIP=0 TOTAL=7
```
<br>

```python
from dbt.cli.main import dbtRunner, dbtRunnerResult

# initialize
dbt = dbtRunner()

# create CLI args as a list of strings
cli_args = ["run", "--select", "tag:my_tag"]

# run the command
res: dbtRunnerResult = dbt.invoke(cli_args)

# inspect the results
for r in res.result:
    print(f"{r.node.name}: {r.status}")
```

<!-- Slide 5 -->
---
layout: two-cols
---

# The first steps

- **Regular manual runs**
- **Cron**
- **CI/CD tooling schedulers**
  - _GitHub Actions, CircleCI, Jenkins_
- **Basic cloud schedulers**
  - _ECS scheduled tasks, GCP Cloud Scheduler, Azure Batch Scheduler_

::right::
<img src="https://i.imgur.com/tm7yfxo.png"/>

<br>
```yml
# .circleci/config.yml
jobs:
  run-dbt:
    steps:
      - checkout
      - run: dbt build

workflows:
  daily-run-dbt:
    jobs:
      - run-dbt:
          triggers:
            - schedule:
                cron: "0 0 * * *"
```

---
layout: statement
level: 2
---

<style>
h1, h3 {color: #2B90B6;}
</style>

# _Checkpoint_

## When have you outgrown basic schedulers?

<br><br>

### _Some ideas:_

You run a lot of jobs for a lot of the day

You have multiple jobs running at once

Your jobs have complex dependencies

Your platform team says CircleCI costs too much

---
layout: two-cols
---

# Orchestrators

- **For more complex scheduling**
  - Running more than just dbt
  - Managing multiple dbt runs
  - Coupling your ingestion with dbt
  - Other ways to start jobs (i.e. sensors)
  - Define your jobs in Python
- **We’re going to talk about Airflow, but there’s alternatives**
  - dbt Cloud
  - Dagster
  - Prefect
  - Newer: Mage, Argo
  - Older: Luigi, Oozie, Azkaban

::right::

<img src="https://airflow.apache.org/docs/apache-airflow/stable/_images/graph.png" />
<br>
<img src="https://raw.githubusercontent.com/astronomer/astronomer-cosmos/main/docs/_static/jaffle_shop_task_group.png" />

<p style="text-align: center">
  <a href="https://pixelastic.github.io/pokemonorbigdata/">Is it Pokemon or Big Data?</a>
</p>
---
layout: default
---
# Starting out with Airflow via the BashOperator

```python {all|10,15|18|all} {lines: true}
with DAG(
    dag_id="example_bash_operator",
    schedule="@hourly",
    start_date=datetime.datetime(2023, 8, 10),
    catchup=False,
) as dag:

    run_dbt = BashOperator(
        task_id="run_dbt",
        bash_command="dbt run --profile prod",
    )

    test_dbt = BashOperator(
        task_id="test_dbt",
        bash_command="dbt test --profile prod",
    )

run_dbt >> test_dbt
```

_"Starting today, run `dbt run`, then `dbt test` once an hour at the top of the hour"_

---
layout: two-cols
---

# BashOperator issues

- **Requirements whack-a-mole**
  - ☹️ [Official AWS MWAAA guide about working around conflicting dependencies](https://docs.aws.amazon.com/mwaa/latest/userguide/samples-dbt.html)
- **File writing conflicts**
  - dbt artifacts + target + logs can be overwritten by concurrent DAG runs - or not written at all
    - Differs between managed Airflow services
  - 💭 If redirecting to `/tmp`, clean it up!
- **Lifecycle conflicts**
  - Upgrading dbt involves requirements changes for your entire Airflow instance
  - In some managed services this can mean downtime


::right::
<img src="https://i.imgur.com/eskvZ9j.png"/>

---
layout: statement
level: 2
---

<style>
h1, h3 {color: #2B90B6;}
</style>

# _Checkpoint_

## When have you outgrown the bash operator?

<br><br>

### _Some ideas:_

You are in dependency hell

You are in race condition hell

---
layout: two-cols
---

# What now?

- **Get hackin’**
  - `PythonVirtualenvOperator`
  - Call a full script from the BashOperator or PythonOperator
    1. Create a venv
    2. Install dependencies to that venv
    3. Run dbt
    4. Clean up*
- **Get containerizin’**
  - Isolate dbt from Airflow
  - Trade code complexity for infra complexity
  - Give Airflow much less to do

::right::
```python {all|3-12|14-16|18-19|21-22|26|all} {lines: true}
base_script = """
    set -e
    # 1) make a venv namespaced with the task
    RUN_ID={{ task.task_id ~ '_' ~ run_id }}
    CLEAN_RUN_ID=$(echo $RUN_ID | tr :+. _)

    mkdir /tmp/$CLEAN_RUN_ID
    DBT_VENV_DIR=/tmp/$CLEAN_RUN_ID/dbt_venv

    /usr/bin/python3 -m virtualenv \
      --python /usr/bin/python3 \
      --creator venv --always-copy $DBT_VENV_DIR

    # 2) Install dependencies to that venv
    $DBT_VENV_DIR/bin/pip3 install -r \
      $DBT_PROFILES_DIR/dbt_requirements.txt

    # 3) run dbt - dbt command envvar set in operator
    $DBT_VENV_DIR/bin/dbt $DBT_COMMAND

    # 4) clean up
    rm -rf $DBT_VENV_DIR
"""
run = BashOperator(
  bash_command=base_script,
  env={"DBT_COMMAND" = "dbt run --profile prod"}
)
```

---
layout: two-cols
---

# Containerization Basics

- `DockerOperator`, `ECSOperator`, or `KubernetesPodOperator`
  - Each isolate your tasks from others
  - Each require some infra work outside Airflow (K8s cluster, ECS cluster, container registry)
- In general - why containerize?
  - Scalability - vertically and horizontally
  - Isolation
- Downsides
  - Complexity
  - In the context of Airflow, local dev gets even more difficult

::right::
<img src="https://i.imgur.com/iJ1mUZU.png" />
---
layout: default
---
# Running dbt via the ECSOperator

```python {all|14} {lines: true}
with DAG(
  # same as before
) as dag:
    run = ECSOperator(
        task_id="run",
        dag=dag,
        cluster="dbt-airflow-cluster",
        task_definition="dbt-airflow-task-def",
        launch_type="FARGATE",
        overrides={
            "containerOverrides": [
                {
                    "name": "ecs-airflow-dbt-task",
                    "command": [f"dbt run --profile prod"],
                }
            ],
        },
        network_configuration={"YOUR_NETWORK": "CONF_GOES_HERE"},
    )
```

_Not pictured: a lot of Terraform work_

---
layout: statement
level: 2
---

<style>
h1, h3 {color: #2B90B6;}
</style>

# _Checkpoint_

## When have you outgrown this?

<br><br>

### _Some ideas:_

You maybe shouldn't have been here to begin with

You have outgrown Airflow, dbt, or batch processing

---
layout: statement
---

# What we did at LaunchDarkly

<!-- Todo: make diagram flow through subsequent slides
Include year
Include dbt version

Audit dbt model
 -->

<br>

```mermaid
%%{init: {'theme':'dark'}}%%
flowchart LR
   A[<b>2018</b> \n Manual runs] -- Growth
   --> B[<b>2019</b> \n BashOperator] -- Dependency hell
   --> C[<b>2020</b> \n Custom plugin] -- AWS migration
   --> D[<b>2022</b> \n Lift and shift] -- Race conditions
   --> E[<b>2023</b> \n ECS + Containers]
```

---
layout: statement
level: 2
---

# The early days

_Disclaimer: I wasn’t actually here for most of this_

```mermaid
%%{init: {'theme':'dark'}}%%
flowchart LR
   A[Manual runs \n Weekend inactivity \n Short dbt runtimes]:::active
   -- "This went exactly as you expect it would"
   --> B[Moved to Airflow 1 on GCP Composer \n BashOperator first]:::active
   -- "Dependency hell"
   --> C[Custom plugin \n Created venvs in /tmp]:::active
   --> D[Happy days on GCP, AWS migration looming]:::inactive
    classDef active fill:teal
    classDef inactive opacity:50%,stroke-dasharray: 5 5
```

📆 _2019-2021_

📚 _20-50 models_

⌚️ _\<10m total runtime_

🤓 _1-3 devs_

🎫 _2-3 deploys per week_

---
layout: statement
level: 2
---

# The GCP days

```mermaid
%%{init: {'theme':'dark'}}%%
flowchart LR
   A[Manual runs, BashOperator]:::inactive
   --> B[Python plugin mostly 'just works' \n SLAs start existing \n Team grows \n SLAs get tighter]:::active
   -- "IT procures an Apple Silicon MBP"
   --> C[We have to upgrade dbt, plugin falls apart \n Team grows some more \n Hourly dbt DAG introduced \n Deploys start becoming a giant pain]:::active
   -- AWS has a managed Airflow offering now
   --> D["Cloud migrations 😰"]:::inactive
    classDef active fill:teal
    classDef inactive opacity:50%,stroke-dasharray: 5 5
```

📆 _2021-2022_

📚 _200+ models_

⌚️ _4h total runtime_

🤓 _4-6 devs_

🎫 _15+ deploys per week_

---
layout: statement
level: 2
---

# The AWS migration

```mermaid
%%{init: {'theme':'dark'}}%%
flowchart LR
   A[GCP is so 2021]:::inactive
   --> B[The Python plugin and GCP no longer 'just works' \n Our architecture did not grow with our team \n LaunchDarkly is not a GCP shop]:::active
   -- "Lift and shift - what could go wrong?"
   --> C[Plugin moved to MWAA, things largely fine \n As we gradually cutover DAGs and load, things stop being fine \n We stare into the abyss and learn how Airflow actually works]:::active
   -- We give up on the lift and shift
   --> D[dbt operations moved to ECS Fargate \n Deploys fully automated \n Plugin deleted]:::active
    classDef active fill:teal
    classDef inactive opacity:50%,stroke-dasharray: 5 5
```

📆 _2022-2023_

📚 _600+ models_

⌚️ _6h total runtime_

🤓 _6-8 devs_

🎫 _20+ deploys per week_

---
layout: statement
level: 2
---
# Today

```mermaid
%%{init: {'theme':'dark'}}%%
flowchart LR
   A[We stared into the abyss]:::inactive
   -- "The abyss stared back"
   --> B[Overall, we're happier \n We should have automated deploys earlier \n Some extra complexity intervening on running tasks \n dbt + dependency upgrades much more manageable]:::active
   --> C[We need to take advantage of this work]:::inactive
    classDef active fill:teal
    classDef inactive opacity:50%,stroke-dasharray: 5 5
```

📆 _2023+_

📚 _600+ models_

⌚️ _2.5h total runtime_

🤓 _7 devs_

🎫 _20+ deploys per week_

---
layout: statement
---

# Is this… good?

<br>
😬 I think this is the most scalable way to run dbt on Airflow

🙂 We’re happy with it for our workload

🫠 It feels more complex than it needs to be

🐙 Airflow is probably not the executor of the future

🚦 SLAs are all green, and we have plenty of headroom to scale

---
layout: end
---

# Thank you!

<br><br><br><br>
## Contact:

[linkedin/deanverhey](https://www.linkedin.com/in/deanverhey/)

[github/verhey](https://github.com/verhey)

[Slide source code](https://github.com/verhey) # todo: update!
