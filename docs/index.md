- [Cesium Scheduler](#cesium-scheduler)
- [What use cases does Cesium Scheduler solve?](#what-use-cases-does-cesium-scheduler-solve)
- [Core Concepts](#core-concepts)
  - [Deployment Model](#deployment-model)
  - [Entities](#entities)
- [Core Entities](#core-entities)
  - [Workspace](#workspace)
  - [Task Executor](#task-executor)
  - [Workflow](#workflow)
    - [Workflow Definition](#workflow-definition)
    - [Task Types](#task-types)
      - [BashTask](#bashtask)
      - [PythonTask](#pythontask)
    - [Workflow Run](#workflow-run)
    - [Lifecyle of a workflow run](#lifecyle-of-a-workflow-run)
- [Workflow Definition Examples](#workflow-definition-examples)
  - [A simple workflow with 1 bash task](#a-simple-workflow-with-1-bash-task)
  - [A simple workflow with 2 tasks with a dependency](#a-simple-workflow-with-2-tasks-with-a-dependency)
- [Task Executor (Tex)](#task-executor-tex)
  - [Requirements](#requirements)
    - [Restrictions](#restrictions)
  - [Running Tex](#running-tex)
  - [Downloading Tex](#downloading-tex)

# Cesium Scheduler

This repository provides code examples and documentation for Cesium Scheduler, a new age recurring tasks scheduler architected for the cloud.

# What use cases does Cesium Scheduler solve?
Cesium Scheduler is a tool that helps you run recurring workflows which are supposed to run at specific time intervals.
Examples include running workflows for data extraction, data movement (using FTP or S3 etc), data processing, periodic restacking, security scans etc.

# Core Concepts

This section explains some of the key things required to work with Cesium Scheduler.

## Deployment Model

Cesium Scheduler is designed to run purely as a SaaS service that is run and managed for you. This allows you to focus on using Cesium to solve key business problems while leaving the operational aspects of running the service and the database to us. The cloud component is sometimes refered to as `Cesium core`, `Cesium Cloud` or `core scheduler` in the documentation.


![High Level Deployment Model](images/cesium-high-level-deployment-model.jpeg)

To run tasks and workflows inside your datacenter or cloud infrastructure, we do not require you to open any ports in your network or firewall. Cesium uses messages and queues to communicate commands and statuses across different systems.

The user needs to download a [Task Executor](#task-executor) and run it as a daemon process on any system in your infrastructure. The tasks and workflows will be executed on this machine by the task executor.


## Entities

The core entities in Cesium are:

- [Workspace](#workspace): The workspace is a way of organizing task executors and workflows
- [Task Executor](#task-executor): A piece of software that needs to be downloaded and run wherever the workflows need to be executed.
- [Workflow](#workflow): An entity which contains a directed acyclic graph of tasks where each task presents a specific step that needs to be run as an independent process and some metadata around alerting and environment varialbes.

# Core Entities

## Workspace

The workspace is a way of organizing task executor and workflow into logical groups for easier management.
If you have multiple teams within your company using Cesium , then you can try to create a workspace per team.
Or if you are running workflows for different purposes, you can group them by purpose.

## Task Executor

The task executor (sometimes also refered to as `Tex` in the documentation) is a software that must be downloaded and run on your infrastructure. The task executor:

- runs as a daemon process within the customer's infrastructure
- is configured through a properties file to authenticate itself to the Cesium Scheduler cloud component.
- is responsible for executing the worfkflow and communicating the state of execution

Every task executor must be associated with a workspace. The task executor requires internet access to work correctly. The task executor initiates outbound network connections from the machine it is running on to the core scheduler in the cloud. Tex never accepts incoming requests from the internet and does not require you to open any ports in your network.

## Workflow

The workflow is the most important entity in Cesium.
When setting up the workflow you must input:

- workspace: The workspace inside which the workflow lives
- the task executor: The tex where the workflow will be executed. You can only associate a workflow with a tex from the same workspace.
- The cron schedule: The cron schedule on which to execute the workflow. The cron is interpretted in UTC timezone.
- Code artifact: The code artifact is a zip file that contains all the files that your workflow needs. This is an optional file. If you attach a code artifact zip file, then the contents will be unzipped into the a specific folder on the machine where the task executor is running and this will be the current directory for the processes launched for the workflow.
- Workflow Definition: The workflow definition is the heart of the workflow. It is a json document that represents the acyclic graph of tasks that must be executed as part of this workflow. The [section below](#workflow-definition) describes the syntax for workflow defintion in more detail.
- Environment Variables: Cesium believes in the methodology of [the 12 factor app](https://12factor.net/codebase) where each workflow can be packaged once and run in different environments using [configs](https://12factor.net/config). You can setup environment variables to point to things like file locations, references to secrets that need to pulled in etc using environment variables. Any variables configured here will be made available to the task process as system or env variables. We strongly discourage using this to pass in secret values. We recommend storing your secrets in a system designed for them like [Consul by HashiCorp](https://www.consul.io/), [AWS Secrets Manager](https://aws.amazon.com/secrets-manager/), or [GCP Secrets Manager](https://cloud.google.com/secret-manager). You can use environment varialbes to store references to the secrets in these secret managers.
- Notifications: This section allows you to get alerted on specific workflow run events like Workflow Run Start, Failure or Success via email.

Once a workflow is configured, Cesium core scheduler creates a [Workflow Run](#workflow-run) and sends out commands to execute the workflow. This happens irrespective of whether the task executor is running or not.
If the task executor is running, it receives the messages and runs the specific workflow. The task executor sends periodic update on the status of the workflow run.
You can also trigger a workflow to run on demand by using the dropdown in the actions column on the workflows page.

### Workflow Definition

The workflow definition is the heart of the workflow. This is a Cesium specific way of representing an acyclic graph of tasks that need to be executed.The worflow definition is expressed in JSON.
Each workflow consists of an attribute called `tasks` which must be an array of Tasks.
Each task consists of a `name` (String), `type` (an enumeration), `depedentTasks` (an array of strings) and then some attributes that are dependent on the task type.

Here is an explanation for each attribute:

|      Name       |       Type     |     Meaning       |
| :-------------: | :------------: |:-------------: |
| `name` (required) |String| A descriptive name for this particular task. This will be shown in the UI in the workflow run details |
| `taskType` (required) |Enumeration| Allowed values are [listed below](#task-types) |
| `depedentTasks` (required) |Array of strings| An array of strings where each entry refers to the `id` of the task that must be run before this task can be run. An empty or null value represents this has no dependencies on any other task. |


[Checkout the examples](#workflow-definition-examples)

### Task Types

The current supported task types are:

#### BashTask

This is used to run bash scripts.
Attributes:
* `command`: the command that needs to be run like `cat`, `/usr/bin/backup_mysql.sh` (which is the command or the bash script that needs to be run). This can be an absolute path to a file already on the machine where tex runs or a relative path to the python script inside the code artifact zip file.
* `args`: an array of strings that are passed to the bash script as parameters.

Examples:
```
{
            "name": "backup",
            "type": "BashTask",
            "command": "/bin/bacup_mysql_db.sh",
            "args": ["production"],
            "dependentTasks": []
    }
```

```
{
        "name": "task1",
        "type": "BashTask",
        "command": "echo",
        "args": ["hello world"]
    }
```

#### PythonTask

This task type is used to run python scripts. Tex currently only supports running python3 scripts. If your task requires specific python libraries, you can define a `requirements.txt` in the code artifact zip file which lists all dependencies in the standard format.
Tex will create a virtual env for this workflow which live inside the workflow folder. The virtual env will be activated each time this task needs to be executed.

The attributes of a python task are: 
* `script`: The script to the python script that needs to be executed. This can be an absolute path to a file already on the machine where tex runs or a relative path to the python script inside the code artifact zip file.
* `args`: an array of strings that are passed to the bash script as parameters.

Examples:
```
    {
        "name": "task2",
        "type": "PythonTask",
        "dependsOn" : ["task1"],
        "script": "run.py",
        "args": ["hello world"]
    }
```


### Workflow Run

A workflow refers to a single run of the workflow. A workflow run is created whenever it is time to execute it (based on the configured cron) or it is executed on demand.
Each workflow run will have the following attributes:

- Requested time: The time at which the workflow run was initiated (whether automatic or manual)
- Start time: The time at which the workflow execution was started by tex
- End time: The time at which the last task in the workflow finished execution
- Status: The current status of the task.
- Execution Type: An enumeration of whether it was scheduled or an on-demand execution.

You can view the workflow run of a specific workflow by choosing the `View History` action in the workflows list or by clicking on the `View History` button on the details of a specific workflow.

### Lifecyle of a workflow run

A workflow run goes through the following lifecycle:

- Requested: The workflow run has been scheduled and a request has been sent to `tex` to execute it.
- Started: The workflow execution has been started by tex.
- Failed: The workflow execution failed due to an error.
- Succeeded: The workflow execution completed successfully.

# Workflow Definition Examples

Here are some examples to help get started.

## A simple workflow with 1 bash task

If you just need a single bash script to be run here is an example.

```
{
    "tasks": [
        {
            "name": "backup",
            "type": "BashTask",
            "command": "/bin/bacup_mysql_db.sh",
            "args": "production",
            "dependentTasks": []
        }
    ]
}
```

## A simple workflow with 2 tasks with a dependency


```
{
    "tasks": [
        {
            "name": "extract",
            "type": "BashTask",
            "command": "extract_database_to_csv.sh",
            "args": "production",
            "dependentTasks": []
        },
        {
            "name": "load",
            "type": "PythonTask",
            "script": "load_to_redshift.py",
            "args": [
                "--env=prod"
            ],
            "dependentTasks": [
                "extract"
            ]
        }
    ]
}
```

# Task Executor (Tex)
The Task Executor is the component of Cesium Scheduler that is deployed inside the customers infrastructure and actually executes the worklfows and manages the processes and outcomes associated with the workflow tasks.
You must download Tex (link below) and run it on your machine.

## Requirements

Following are the requirements for running Tex:
* Tex is designed to run on JRE 1.8 (any JVM > 1.8 is good enough)
* Tex does not need to be run as root.
* The JVM will consume about 512 MB of RAM
* The machine where tex runs requires outbound internet action to reach Cesium Scheduler's cloud servers.
* Tex will not open any inbound ports on your machine and does not require changes to inbound rules on your firewall.
* Tex requires bash to be available if there are bash tasks that need to be executed
* Tex requires Python 3 and pip to be available if there are Python tasks to be executed.

### Restrictions

Restrictions: 
* Tex does not work on Windows environments and will only run on nix systems
  


## Running Tex
Use the following steps to run tex:
1. Download the tex zip using the link on the download tex page
2. Optionally create a user for tex with non-super user status.
3. Unzip the contents into the location where you want to run tex like `/opt/cesium/tex`.
4. Once the contents are unzipped into a folder, we will refer to this folder as `TEX_HOME`.
5. Go to the web console of Cesium Scheduler and navigate to the specific task executor you are trying to run. There will be a button there to download the config file for this tex. Press it and save the file locally.
6. Under `TEX_HOME` there is a folder called config with a single config file under it called `tex-config.properties`. Update the config values from the values you get from the file you downloaded earlier.
7. You must now have a filled up config file that will look like this:
```
baseURL: app.cesiumscheduler.com
texId: ahj721haasea
password: ek12313y1UwXpMd7PzSUW4OT9
tenantRefId: 9abcd31l-1234-5678-9012-286dd73aec45
useTLS: true
heartBeatPeriod: 60
```
8. Run the start script: `$TEX_HOME/bin/start-cesium-tex.sh` . This will start tex as a background process and will write out logs under the logs folder. You can safely exit the user and the shell where you started the tex instance.

## Downloading Tex

You can download tex from [this page](download-tex.md).
