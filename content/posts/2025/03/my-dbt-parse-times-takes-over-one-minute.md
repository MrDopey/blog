--- 
date: 2025-03-26T08:58:19Z
title: 'My Dbt Parse Times Takes Over One Minute'
description: ""
slug: ''
authors: []
tags: [ 'dbt', 'google-cloud-composer', 'air-flow', 'medallion-architecture', 'ETL', 'descent-into-madness', 'claude', 'llm-coding' ]
categories: []
externalLink: ""
series: []
---

## Background

On a project that I have worked on, we are using [Google Cloud Composer](https://cloud.google.com/composer/docs/composer-2/composer-overview) (which is really a managed [airflow](https://github.com/apache/airflow) service) to run an ETL pipeline. We used [dbt](https://docs.getdbt.com/) to wangle the large amounts of sql scripts and to auto generate a dag, to then be orchestrated by airflow, which is glued together by [Astronomer Cosmos](https://astronomer.github.io/astronomer-cosmos/index.html).

The issue is that the airflow scheduler periodically parse our dags, and when the parsing time > period, then bad things start to happen. For example, the schedulers enters a boot loop, as it does not think the scheduler is running correctly.

Cloud composer 2 has `dagbag_import_timeout` = 120 seconds (docs currently list cloud composer 3) <https://cloud.google.com/composer/docs/airflow-configurations>

Default airflow timeout = 30 seconds <https://airflow.apache.org/docs/apache-airflow/2.6.3/configurations-ref.html#dagbag-import-timeout>

According to [airflow docs](https://airflow.apache.org/docs/apache-airflow/stable/tutorial/fundamentals.html#it-s-a-dag-definition-file) the expected time to parse dags should be measured in seconds, emphasis mine
> People sometimes think of the DAG definition file as a place where they can do some actual data processing - that is not the case at all! The script’s purpose is to define a DAG object. **It needs to evaluate quickly (seconds, not minutes)** since the scheduler will execute it periodically to reflect the changes if any.

## My descent into madness

### Reproducing the error

My first thought was to reproduce the issue with a minimal example.

To keep this it simple, the dag is pretty straight forward, the tasks are chained liked [so](https://airflow.apache.org/docs/apache-airflow/stable/tutorial/fundamentals.html#setting-up-dependencies) where each layer of the medallion is a [Cosmos DbtTaskGroup](https://www.astronomer.io/docs/learn/airflow-dbt/)

```python
ingestion = TaskGroup(....)
bronze_taskgroup = DbtTaskGroup(....)
silver_taskgroup = DbtTaskGroup(....)
gold_taskgroup = DbtTaskGroup(....)
egress = TaskGroup(....)

(
  ingestion
  >> bronze_taskgroup
  >> silver_taskgroup
  >> gold_taskgroup
  >> egress
)
```

For some reason I couldn't reproduce the error locally from my machine (more on that later), I had to run it in a live cloud composer environment, our testing environment, and the results were interesting

The only combination that caused the dags to parse for a long time was

```python
(
  bronze_taskgroup
  >> silver_taskgroup
)
# or
(
  silver_taskgroup
  >> bronze_taskgroup
)
# Which didn't make sense logically, but an interesting data point regardless
```

### Red herring # 1 - dbt parsing

I did some deep, deep research into the [dbt parsing](https://docs.getdbt.com/reference/parsing) and tried a few different things with partial parsing (using `dbt parse` cli command).

Locally on my machine, the cache is being hit, and I understood how it works, but no idea how that information translates into the a running cloud composer environment.

Upon further digging, it turns out [cosmos parsing](https://astronomer.github.io/astronomer-cosmos/configuration/parsing-methods.html) was the relevant information I was missing (well not missing, I've read it a few times by now and didn't make the connection).

The specific parsing method we use is `dbt_ls_file`, which means dbt has already parsed and compiled everything we needed already, so there really isn't anything to 'parse' from the tool dbt. All that is expected from the ETL is the pregenerated `dbt ls` files that is actually run during the CI/CD process.

The point is, all the heavy lifting from dbt is already done before it reaches the cloud composer environment and I need to look elsewhere for my bug.

### Detour # 1 - Reproducing the error locally with local instance of cloud composer

The not being able to reproduce the error was starting to bug me, I really wanted to know _why_ I couldn't reproduce the bug, as far as I can tell my environment is setup the same as cloud composer.

At this point I came across pretty much exactly what I am looking for

- <https://cloud.google.com/composer/docs/composer-2/run-local-airflow-environments>
- <https://github.com/GoogleCloudPlatform/composer-local-dev/tree/0.9.2>

It was a very fiddly and buggy process, and it took the good part of a day

- my sqlite database kept getting locked out and I needed to delete the sqlite file and restart the local cloud composer instance
- lots of permission issues, so needing to run as root
- had lots of issues when I stopped the instance and started it again, so it was easier to rebuild the whole thing, taking minutes to be able to use it again (but still faster than a live instance)

I was finally able to reproduce the error, and iterate much faster.

### Detour 2 - The source code

I had a looky loo, at the source code for `>>` in dbt.

1. The task mixin class implements [`__rshift__`](https://github.com/apache/airflow/blob/eb24742d5300d2d87b17b4bcd67f639dbafd9818/airflow/models/taskmixin.py#L85)
1. which calls [`self.set_downstream`](https://github.com/apache/airflow/blob/eb24742d5300d2d87b17b4bcd67f639dbafd9818/airflow/models/taskmixin.py#L223)
1. which calls [`set_relatives`](https://github.com/apache/airflow/blob/eb24742d5300d2d87b17b4bcd67f639dbafd9818/airflow/models/taskmixin.py#L165)
1. which calls [`TaskGroup.get_leaves`](https://github.com/apache/airflow/blob/eb24742d5300d2d87b17b4bcd67f639dbafd9818/airflow/utils/task_group.py#L364)

And then I lost all interest, as it didn't seem to lead anywhere interesting.

The only thing that could make any sense is the number of tasks in each TaskGroup, as some of combinatorial explosion problem, but I'm not familiar enough with `dbt` source to even have a stab at where to start.

I dug a bit into my files

- Bronze: 3500 tasks
- Silver: 1600 tasks
- Gold: 50 tasks

Since I now have a locally running instance of cloud composer, I ran a battery of test, by truncating the `dbt ls` file of varying amounts of tasks, e.g. 2000, 1000, 500, 100 tasks and there was a strong correlation between reducing number of tasks and reducing the parsing time. However, I have no idea what causes this, and how to fix the underlying cause.

### Red herring # 2 - configuration changes

I got really intimate with the airflow configuration and troubleshooting documentation for, literally, everything I could find

- <https://cloud.google.com/composer/docs/composer-2/troubleshooting-dags#task-fails-without-logs-dag-errors>
- <https://cloud.google.com/composer/docs/composer-2/troubleshooting-scheduling#troubleshooting_issues_at_dag_parse_time>
- <https://airflow.apache.org/docs/apache-airflow/2.6.3/configurations-ref.html>
- <https://airflow.apache.org/docs/apache-airflow/2.6.3/best-practices.html>
- <https://astronomer.github.io/astronomer-cosmos/configuration/partial-parsing.html>
- <https://airflow.apache.org/docs/apache-airflow-providers-google/stable/index.html>

I tried everything that was remotely looked like part of the problem of parsing dbt files, but nothing worked.

The good thing was that I was able to fail very fast, since I'm now running a local instance of cloud composer. Testing in an actual cloud composer environment is a pain when changing configuration values, because the environment is [terraformed](https://www.terraform.io/), and you need to bug someone on another team to approve and deploy the changes on your behalf :vomiting_face:.

### Detour # 3 - Using LLMs

So we have this shiny new tool at work called [Claude](https://claude.ai/). I managed to coerce it offer suggestions, which I tried, and it didn't help.

#### Detour # 3.1 - Celery Queues

So Claude was really insistent on celery queues, which I did read about previously when throwing configuration changes at the wall, but discarded it because it didn't seem relevant.

So down another rabbit hole I went, and all I learnt is that [Celery](https://docs.celeryq.dev/en/stable/) is about task management and didn't really affect the parsing of the dag.

#### Detour # 3.2

After a few hours of testing Claude's suggestion and trying to coerce it to come up with alternate solution, I come out empty handed.

In retrospect, Claude is doing two things

- Answering exactly what I am asking it, trying to fix long dag parse times (spoiler, that wasn't the right question to ask)
- Regurgitating exactly what it was trained on, the documentation, all of which I've already read, digested, and discarded through testing and my understanding of the system I am using.

### Detour 4 - local cloud composer != local machine

At this point in time I wanted tackle the problem at a different angle, to see if I could get new information, as I have; what I believe to be, learnt everything there is to know about parsing, airflow, cloud composer and troubleshooting ... you get the point.

I wanted to answer why my local instance of cloud composer (running in a docker container) is reproducing the error but not my local laptop environment in a simple virtual python environment.

According to the [documentation](https://airflow.apache.org/docs/apache-airflow/2.6.3/best-practices.html#dag-loader-test) I can test the dag parsing by running `python ./script_with_dag.py`, but my parsing time was consistently at sub 1 second, but the local cloud composer instance was parsing at 60 seconds (yes this is shorter than the running instance of cloud composer at 120 seconds, but I'm assuming they are both the same underlying issue).

I'm starting to doubt that that statement to test local parsing is accurate.

I opened an interactive shell session with my local cloud composer instance and ran `python ./script_with_dag.py`
> results: 60 seconds

![skeptical](https://media1.tenor.com/m/s1wnF2DiWA0AAAAd/skeptical-futurama.gif)

Okay, something is up.

I started comparing python packages from inside the local cloud composer instance and my virtual environment, just by running `pip freeze`, sort list, and using a text diff-tool and comparing the two list.

After various effort of upgrading and downgrading my virtual environment to slowly match the local composer instance, and running `python ./script_with_dag.py`, I strike gold (and in hindsight I should have picked out important looking packages first), parsing time is now 60 seconds in my virtual python environment.

The culprit? My virtual environment was on airflow version 2.10.0 (the latest version at time of testing) versus the cloud composer instance at version 2.6.3

## The root cause

So I read the entire [release notes](https://airflow.apache.org/docs/apache-airflow/stable/release_notes.html#airflow-2-7-0-2023-08-18) (fortunately it was only 1 version difference between 2.6.3 and 2.7.0) to really understand was the root cause.

I found this little gem

> Optimize performance of scheduling mapped tasks ([#30372](https://github.com/apache/airflow/pull/30372))

Which sounds exactly what I have observed.

## The solution

There was only 3 options that I could think of to resolve this

- Upgrade to airflow 2.7.0 or above, we had issues upgrading versions before (technically you need to upgrade cloud compose which indirectly upgrades airflow) and we were looking for a short term fix
- Smoosh the bronze, silver, gold tasks together - which we didn't want to do, as the delineation of the task group makes it easier for the Operations teams to figure out where progression in the ETL
- The issue only occurs when 2 large dbt groups are adjacent to each other, so adding a no-op task between them should suffice

We went with the last option

```python
bronze_silver_buffer = EmptyOperator(task_id="bronze_silver_buffer)
...
(
  ingestion
  >> bronze_taskgroup
  >> bronze_silver_buffer
  >> silver_taskgroup
  >> gold_taskgroup
  >> egress
)

```

Also, pinning the dam `requirements.txt` from our repo to airflow 2.6.3.

## What did I learn?

LLMs would get interesting if it can include your runtime as part it's context and to help debug; also terrifying, imagine an LLM having write access to your production environment :scream:.

There's a reason why pinning your dependencies is recommended.