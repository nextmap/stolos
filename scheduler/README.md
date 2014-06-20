Summary:
==============

This task scheduler manages the execution of tasks where some tasks may
depend on prior completion of other tasks.  It has the following
features:

  - very simple to use (create an issue if it isn't!)
  - simultaneous and fault tolerant execution of tasks (via ZooKeeper)
  - characterize dependencies in a directed acyclic multi-graph
  - support for subtasks and dependencies on arbitrary subsets of
    subtasks
  - language agnostic
  - "at least once" semantics (a guarantee that a task will successfully
    complete or fail after n retries)
  - designed for tasks of various sizes: from large hadoop tasks to
    tasks that take a second to complete

What this is project not:
  - not meant for "real-time" computation
  - unaware of machines, nodes, network topologies and infrastructure
  - does not (and should not) auto-scale workers
  - by itself, it does no actual "work" other than managing job state
    and queueing future work


Requirements:
  - ZooKeeper
  - Some Python libraries (Kazoo, Networkx, Argparse, ...)

Optional requirements:
  - Apache Spark
  - GraphViz


Background: Inspiration
==============


The inspiration for this project comes from the notion that the way we
manage dependencies in our system defines how we characterize the work
that exists in our
system.

Sailthru's Data Science team has a complex data pipeline and set of
requirements.  We have to build models, algorithms and data pipelines
for many clients, and this leads to a wide variety of work we have to
perform within our system.  Some work is very specific.  For instance,
we need to train the same predictive model once per (client, date).
Other tasks might be more complex: a cross-client analysis across
various groups of clients and ranges of dates.  In either case, we
cannot return results without having previously identified data sets we
need, transformed them, created some extra features on the data, and
built the model or analysis.

Since we have hundreds or thousands of instances of any particular task,
we cannot afford to manually verify that work gets completed. Therefore,
we need a system to manage execution of tasks.


Concept: Task Dependencies as a Directed Graph
==============


We can model dependencies between tasks as a directed graph, where nodes
are tasks and edges are dependency requirements.  The following section
explains how this scheduler uses a directed graph to define task
dependencies.

We start with an assumption that tasks depend on each other:

           Scenario 1:               Scenario 2:

              Task_A                     Task_A
                |                        /     \
                v                       v       v
              Task_B                  Task_B   Task_C
                                        |       |
                                        |      Task_D
                                        |       |
                                        v       v
                                          Task_E


In Scenario 1, `Task_B` cannot run until `Task_A` completes In Scenario
2, `Task_B` and `Task_C` cannot run until `Task_A` completes, but
`Task_B` and `Task_C` can run in any order.  Also, `Task_D` requires
`Task_C` to complete, but doesn't care if `Task_B` has run yet.
`Task_E` requires `Task_D` and `Task_B` to have completed.

We also support the scenario where one task expands into multiple
subtasks.  The reason for this is that we run a hundred or thousand
variations of the one task, and the results of each subtask may bubble
down through the dependency graph.

There are several ways subtasks may depend on other subtasks, and this
system captures them all (as far as we can tell).


Imagine the scenario where `Task_A` --> `Task_B`


            Scenario 1:

               Task_A
                 |
                 v
               Task_B


Let's say `Task_A` becomes multiple subtasks, `Task_A_i`.  And `Task_B`
also becomes multiple subtasks, `Task_Bi`.  Scenario 1 may transform
into one of the following:


###### Scenario1, Situation I

     becomes     Task_A1  Task_A2  Task_A3  Task_An
     ------->      |          |      |         |
                   +----------+------+---------+
                   |          |      |         |
                   v          v      v         v
                 Task_B1  Task_B2  Task_B3  Task_Bn

###### Scenario1, Situation II

                 Task_A1  Task_A2  Task_A3  Task_An
     or becomes    |          |      |         |
     ------->      |          |      |         |
                   v          v      v         v
                Task_B1  Task_B2  Task_B3  Task_Bn


In Situation 1, each subtask, `Task_Bi`, depends on completion of all of
`TaskA`'s subtasks before it can run.  For instance, `Task_B1` cannot
run until all `Task_A` subtasks (1 to n) have completed.  From the
scheduler's point of view, this is not different than the simple case
where `Task_A(1 to n)` --> `Task_Bi`.  In this case, we create a
dependency graph for each `Task_Bi`.  See below:

###### Scenario1, Situation I (view 2)

     becomes     Task_A1  Task_A2  Task_A3  Task_An
     ------->      |          |      |         |
                   +----------+------+---------+
                                 |
                                 v
                               Task_Bi


In Situation 2, each subtask, `Task_Bi`, depends only on completion of
its related subtask in `Task_A`, or `Task_Ai`.  For instance, `Task_B1`
depends on completion of `Task_A1`, but it doesn't have any dependency
on `Task_A2`'s completion.  In this case, we create n dependency graphs.

As we have just seen, dependencies can be modeled as directed acyclic
multi-graphs.  (acyclic means no cycles - ie no loops.  multi-graph
means a it contains many separate graphs).  Scenario 2 is the default in
this scheduler.


Concept: Job IDs
==============

*For details on how to use and configure `job_id`s, see the section, "Job
ID Configuration"*  # TODO

The scheduler recognizes tasks (ie `TaskA` or `TaskB`) and subtasks
(`TaskA_1`, `TaskA_2`, ...).  A task represents a group of similar
`job_id`s.  A `job_id` identifies subtasks, and it is made up of
"identifiers" that we mash together into a `job_id` template.


Some example `job_id`s and their corresponding template might look like
the example below:

    "20140614_client1_dataset1"  <------>  "{date}_{client_id}_{dataset}"
    "20140601_analysis1"  <------>  "{date}_{your_custom_identifier}"


A `job_id` represents the smallest piece of work the scheduler can
recognize, and good choices in `job_id` structure identify how work is
changing from task to task.  For instance, assume the second `job_id`
above, `20140601_analysis1`, depends on all `job_id`s from 20140601 that
matched a specific subset of clients and datasets.  We chose to identify
this subset of clients and datasets with the name "analysis1."  But our
`job_id` also includes a date because we wish to run analysis1 on
different days.  Note how the choice of `job_id` clarifies what the first
and second tasks have in common.

Here's some general advice for choosing a `job_id` template:

  - What results does this task generate?  The words that differentiate
    those results are great candidates for identifers in a `job_id`.
  - What parameters does this task expect?  The command-line arguments
    to a piece of code can be great `job_id` identiers.
  - How many different variations of this task exist?
  - How do I expect to use this task in my system?
  - How complex is my data pipeline?  Do I have any branches in my
    dependency tree?  If you have a very simple pipeline, you may simply
    wish to have all `job_id`s be the same across subtasks.


Configuration: Job IDs
==============

This section explains what configuration for `job_id`s must exist.

These environment variables must be set and available to scheduler code:

    export JOB_ID_DEFAULT_TEMPLATE="{date}_{client_id}_{collection_name}"
    export JOB_ID_VALIDATIONS="tasks.job_id_validations"

- `JOB_ID_VALIDATIONS` points to a python module containing code to
  verify the identifiers in a `job_id` are correct.
  - See `scheduler/examples/job_id_validations.py` for the expected
    code structure  # TODO link
- `JOB_ID_DEFAULT_TEMPLATE` - defines the default `job_id` for a task if the
  `job_id` template isn't explicitly defined in the tasks.json.

In addition to these defaults, a task in the tasks.json configuration
may also contain a job_id template.  See "Configuration: Tasks" for
details.  # TODO


Configuration: Tasks
==============

A JSON file defines the task dependency graph and all related
configuration metadata. This section will show available configuration
options.  For instructions on how to use this file, see section TODO!!!!

Here is a minimum viable configuration for a task (api subject to
change):

    {
        "task_name": {
            "job_type": "bash"
        }
    }

Here is a more complex example of a `TaskA_i` --> `TaskB_i`
relationship:

    {
        "task1": {
            "job_type": "bash",
            "bash_opts": "echo Running Task 1 | grep 1"
        },
        "task2": {
            "job_type": "bash",
            "bash_opts": "echo Running Task 2"
            "depends_on": {"app_name": ["task1"]}
        }
    }

And more complex variant of a `TaskA_i` --> `TaskB_i` relationship:

    {
        "preprocess": {
            "job_type": "bash",
            "job_id": "{date}_{client_id}"
        },
        "modelBuild"": {
            "job_type": "bash",
            "job_id": "{date}_{client_id}_{target}"
            "depends_on": {
                "app_name": ["preprocess"],
                "target": "purchase_revenue"
            }
        }
    }

A very complicated dependency graph demonstrating how `TaskA` expands
into multiple `TaskB_i`:

    {
        "preprocess": {
            "job_type": "bash",
            "job_id": "{date}_{client_id}"
        },
        "modelBuild"": {
            "job_type": "bash",
            "job_id": "{date}_{client_id}_{target}"
            "depends_on": {
                "dependency_group_1": {
                    "app_name": ["preprocess"],
                    "target": ["purchase_revenue"]
                },
                "group2": {
                    "app_name": ["preprocess"],
                    "target": ["number_of_pageviews"]
                }
        }
    }

This configuration demonstrates how multiple `TaskA_i` reduce to
`TaskB_j`:

    {
        "preprocess": {
            "job_type": "bash",
            "job_id": "{date}_{client_id}"
        },
        "modelBuild": {
            "job_type": "bash",
            "job_id": "{client_id}_{target}"
            "depends_on": {
                "dependency_group_1": {
                    "app_name": ["preprocess"],
                    "target": ["purchase_revenue"],
                    "date": [20140601, 20140501, 20140401]
                },
                "group2": {
                    "app_name": ["preprocess"],
                    "target": ["number_of_pageviews"],
                    "date": [20140615]
                }
        }
    }


There are other variations of configuration options.  For a complete
list of options, refer to the following table:

- *`root`* - (optional) Label this (optional)de as a root (optional)de
  (meaning it has (optional) parents)
- *`valid_if_or`* - (optional) Criteria that `job_id`s are matched against.
  If a `job_id` for a task does (optional)t match the given
  `valid_if_or` criteria, then the task is immediately marked as
  "completed"
- *`depends_on`* - (optional) A specification of all parent `job_id`s and
  tasks, if any.
- *`job_id`* - (optional) A template describing what identifiers compose
  the `job_id`s.
- *`job_type`* - (required) Select how to execute the following task's
  code.  The `job_type` choice also adds other configuration options


Configuration: Job Types
==============

The `job_type` specifier in the config defines how your application code
should run.  For example, should your code be treated as a bash job (and
executed in its own shell), or should it be an Apache Spark (python) job
that receives elements of a stream or a textFile instance?  The
following table defines different `job_type` options available.  Each
`job_type` has its own set of configuration options, and these are
available at the commandline and probably in the tasks.json file.

 - job_type="bash"
    - bash_options
    - ????
 - job_type="spark"
    - map
    - mapJson
    - textFile
    - read_fp
    - read_s3_key
    - read_s3_bucket
    - write_fp
    - write_s3_bucket
    - ???

You can easily extend this system to support your own custom
applications and functionality by specifying a `job_type`.  As an example,
see the files in THIS GITHUB DIRECTORY for details.  # TODO


Setup:
==============

# TODO: pip install scheduler


This section explains how to use this scheduler

The first time only, you need to create some basic environment vars and
files:

    export TASKS_JSON="/path/to/a/file/called/tasks.json"
    export JOB_ID_DEFAULT_TEMPLATE="{date}_{client_id}_{collection_name}"
    export JOB_ID_VALIDATIONS="tasks.job_id_validations"

- `TASKS_JSON` is the filepath to the json file explained in the
"Configuration" section.  This file describes tasks, their dependencies
and some other basic metadata about how to execute them.
TODO: move this into another section

- **See scheduler/examples/ for details.**
- **See scheduler/bin/testme for example environment var configuration.**

After this initial setup, you will need to create a task and register
it.  This takes the form of these steps:

1. Create some application that can be called through bash or initiated
   as a spark job
1. Create an entry for it in the tasks.json
1. Submit a `job_id` for this task
1. Run the task.

- **See scheduler/examples/tasks/minimal_viable_example.py for details
  on how to perform these simple steps.**

# TODO: insertlink above
# TODO: continyue the readme and decide on sections

Roadmap:
===============

Here are some improvements we are planning for in the near future:

- A web UI for creating, viewing and managing tasks and task dependencies
  - Ability to create multiple dependency groups
  - Interactive dependency graph
- Integration with Apache Marathon