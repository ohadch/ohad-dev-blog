+++
title = 'Scheduling Api Requests With Prefect'
date = 2023-09-10T08:09:53+03:00
draft = true
+++

## Introduction

One of the most common tasks in data engineering is to make API requests. 
This is a task that can be automated with Prefect. 
Prefect is a Python library that allows you to build, schedule and monitor workflows. 
It is a great tool for automating data science tasks. 
In this post, I will show you how to use Prefect to schedule API requests.

## Assumptions

In this post, I will assume that you have a basic understanding of Python. 
You do not need to have any prior knowledge of Prefect. 
I will also assume that you have a basic understanding of APIs. 

In this post, I attempt to demonstrate an end-to-end solution that is simple enough for those who are new to prefect and DAGs.
Hence, I will only cover the basics of each topic.
If you want to learn more about Prefect, I recommend you to check out the [Prefect documentation](https://docs.prefect.io/).

## Understanding DAGs (Directed Acyclic Graphs)

### What is a DAG?

In order to understand Prefect, we need to understand DAGs. DAG stands for Directed Acyclic Graph. 
In the context of workflows, a DAG is a graph that represents the dependencies between tasks. 
Each node in the graph represents a task, and each edge represents a dependency between tasks.

This is how the the three components of DAG stand for:
- Directed: The edges in the graph have a direction. In our example, the direction is from `task1` to `task2`, and from `task2` to `task3`.
- Acyclic: There are no cycles in the graph. Aka, there are no loops in the graph. In our example, if `task1` depends on `task3`, then we have a cycle in the graph.
- Graph: The graph is a collection of nodes and edges. In our example, the nodes are `task1`, `task2`, and `task3`. The edges are the dependencies between the tasks.

### An example DAG

Imagine you're getting ready for a cozy dinner at home with your family.

You have three main tasks: You need to buy groceries, prepare food, and prepare the table. Only then, you can eat.

So a simple DAG for this workflow could be:

```
buy_groceries <- prepare_food <- prepare_table
```

That is a simple DAG. But it can be improved: while the food is being prepared, 
you can ask your significant other to prepare the table, so you can save some time by running these tasks in parallel.

So a better DAG for this workflow could be:

```
buy_groceries <- prepare_food
buy_groceries <- prepare_table
```

In this DAG, `prepare_food` and `prepare_table` tasks depend on `buy_groceries` task.

This is a simple example, but it represents the general idea of DAGs.

### Why are DAGs important?

DAGs are important in workflow management because they provide a structured way to represent task dependencies, 
enabling efficient parallel execution, visualization, and error handling in complex workflows.

This was just a short introduction to DAGs. If you want to learn more about DAGs, stay tuned for my future posts.

Now, let's dive into our use case, and then use Prefect to solve it.

## Our use case

Let's say we are bored, and we would like to get ideas for things to do. 
Fortunately, there is an open source API that provides us with ideas for things to do. 
The API is called [Bored API](https://www.boredapi.com/). It is a simple API that returns a random activity that you can do.

The API has a single endpoint: `https://www.boredapi.com/api/activity`. 
When we make a GET request to this endpoint, we get a JSON response that looks like this:

```json
{
  "activity": "Learn Express.js",
  "type": "education",
  "participants": 1,
  "price": 0,
  "link": "",
  "key": "3943500",
  "accessibility": 0.1
}
```

The response contains the activity, the type of the activity, the number of participants, the price, the link, the key, and the accessibility.

Now, let's say we want to get a random activity every day at 9:00 AM, print it to the console, and save it as a Prefect artifact. 

A sample DAG for this workflow could be as simple as:

```
get_random_activity <- print_activity <- save_artifact
```

In this DAG, `print_activity` depends on `get_random_activity`, and `save_artifact` depends on `get_random_activity`.

We can even run `print_activity` and `save_artifact` in parallel, as they both depend on `get_random_activity`:
    
```
get_random_activity <- print_activity
get_random_activity <- save_artifact
```

Let's see how we can implement this workflow with Prefect.


## Starting a Prefect project

First, let's create a new directory for our project:

```bash
mkdir prefect-api-requests
cd prefect-api-requests
```

Then, let's create a new Python 3.10 virtual environment:

```bash
python3.10 -m venv venv
```

If you do not have Python 3.10 installed, you can install it with [pyenv](https://github.com/pyenv/pyenv).

Next, let's activate the virtual environment:

```bash
source venv/bin/activate
```

Now, create `requirements.txt` file with `prefect==2.13.0` and `requests==2.31.0` as its contents:
    
```bash
echo "prefect==2.13.0" > requirements.txt
echo "requests==2.31.0" >> requirements.txt
```

Then, install the requirements:

```bash
pip install -r requirements.txt
```


## Creating a Prefect flow

The way to create a DAG in Prefect is to create a Prefect flow.

A Prefect flow is a Python function that is decorated with `@flow` decorator. It represents a DAG.
The tasks in the flow are represented by Python functions that are decorated with `@task` decorator.
Finally, the dependencies between the tasks are represented by the function calls in the flow function.


Although we can create a Prefect project with the `prefect` CLI, for this example we will simply create the flow manually.

I like to create a `flows` directory for my flows:

```bash
mkdir flows
```

Then, let's create a new file called `suggest_activity.py` inside the `flows` directory. This file will contain our flow.

First, let's import the `flow` and `task` decorators from the `prefect` library:

```python
from prefect import flow, task
```

Then, let's create a `get_random_activity` task. This task will make a GET request to the Bored API, and return the response:

```python
@task
def get_random_activity() -> str:
    """
    Get a random activity from the Bored API
    """
    import requests
    response = requests.get("https://www.boredapi.com/api/activity")
    return response.json()["activity"]
```

Then, let's create a `print_activity` task. This task will print the activity to the console:

```python
@task
def print_activity(activity: str) -> None:
    """
    Print the activity to the console
    :param activity: The activity to print
    """
    print(f"The activity is: {activity}")
```

And lastly, let's create a `save_artifact` task. This task will save the activity to a Prefect artifact, 
that we can view in the Prefect UI:

```python
@task
def save_activity_artifact(activity: str) -> None:
    """
    Print the activity to the console
    :param activity: The activity to print
    """
    create_markdown_artifact(
        markdown=f"""
        Today, you should do the following activity: **{activity}**. 
        """,
        key="activity"
    )
```

Finally, let's create a `suggest_activity` flow. This flow will call the `get_random_activity` task, and then call the `print_activity` task:

```python
@flow
def suggest_activity() -> None:
    """
    Suggest a random activity
    """
    # First, we need to get a random activity
    activity = get_random_activity.submit()
    
    # Then, we can both print it to the console, and save it as an artifact, but we can do it in parallel
    print_activity.submit(activity)
    save_activity_artifact.submit(activity)
```

In the `suggest_activity` flow, we first call the `get_random_activity` task, 
and then we call the `print_activity` task and the `save_activity_artifact` task in parallel.

This is possible because Prefect is smart enough to figure out the dependencies between the tasks.
As both `print_activity` and `save_activity_artifact` tasks depend directly on the result of the `get_random_activity` 
task (that is, the `activity` variable), Prefect will run them in parallel.

That's it! We have created our flow. This is how the `flows/suggest_activity.py` file should look like:

```python
from prefect import flow, task
from prefect.artifacts import create_markdown_artifact


@task
def get_random_activity() -> str:
    """
    Get a random activity from the Bored API
    """
    import requests
    response = requests.get("https://www.boredapi.com/api/activity")
    return response.json()["activity"]


@task
def print_activity(activity: str) -> None:
    """
    Print the activity to the console
    :param activity: The activity to print
    """
    print(f"The activity is: {activity}")


@task
def save_activity_artifact(activity: str) -> None:
    """
    Print the activity to the console
    :param activity: The activity to print
    """
    create_markdown_artifact(
        markdown=f"""
        Today, you should do the following activity: **{activity}**. 
        """,
        key="activity"
    )


@flow
def suggest_activity() -> None:
    """
    Suggest a random activity
    """
    activity = get_random_activity.submit()
    print_activity.submit(activity)
    save_activity_artifact.submit(activity)


if __name__ == "__main__":
    suggest_activity()
```

Please note that we have added a `if __name__ == "__main__"` block at the end of the file, so that we can run the flow from the command line.
Nevertheless, this is done only for testing purposes. We will not use this block when we run the flow with Prefect. Instead, we will use a Prefect deployment for that.

Let's test our flow by running it from the command line:

```bash
python flows/suggest_activity.py
```

If everything went well, you should see an output similar to this:

```
21:33:40.150 | INFO    | prefect.engine - Created flow run 'liberal-trout' for flow 'suggest-activity'
21:33:40.419 | INFO    | Flow run 'liberal-trout' - Created task run 'get_random_activity-0' for task 'get_random_activity'
21:33:40.420 | INFO    | Flow run 'liberal-trout' - Submitted task run 'get_random_activity-0' for execution.
21:33:40.452 | INFO    | Flow run 'liberal-trout' - Created task run 'print_activity-0' for task 'print_activity'
21:33:40.453 | INFO    | Flow run 'liberal-trout' - Submitted task run 'print_activity-0' for execution.
21:33:40.463 | INFO    | Flow run 'liberal-trout' - Created task run 'save_activity_artifact-0' for task 'save_activity_artifact'
21:33:40.464 | INFO    | Flow run 'liberal-trout' - Submitted task run 'save_activity_artifact-0' for execution.
21:33:41.352 | INFO    | Task run 'get_random_activity-0' - Finished in state Completed()
The activity is: Clean out your garage
21:33:41.417 | INFO    | Task run 'print_activity-0' - Finished in state Completed()
21:33:41.604 | INFO    | Task run 'save_activity_artifact-0' - Finished in state Completed()
21:33:41.637 | INFO    | Flow run 'liberal-trout' - Finished in state Completed('All states completed.')
```

As you can see, the flow ran successfully, and we got a random activity printed to the console.

Now it is a good time to get familiar with the Prefect UI. 

In order to do that, let's start the Prefect server:

```bash
prefect server start
```

If everything went well, you should see an output similar to the following in the CLI:

```
 ___ ___ ___ ___ ___ ___ _____ 
| _ \ _ \ __| __| __/ __|_   _| 
|  _/   / _|| _|| _| (__  | |  
|_| |_|_\___|_| |___\___| |_|  

Configure Prefect to communicate with the server with:

    prefect config set PREFECT_API_URL=http://127.0.0.1:4200/api

View the API reference documentation at http://127.0.0.1:4200/docs

Check out the dashboard at http://127.0.0.1:4200
```

Now, let's open the Prefect UI in our browser by navigating to `http://localhost:4200`.

You should see a screen similar to this:
![Prefect UI](/posts/scheduling-api-requests-with-prefect/prefect-dashboard.png)


This is the Prefect UI. Currently, we see the dashboard page, 

Inside the `Flow Runs` box, we see flow runs broken down by state. 
By default, we see the flow runs that are in `Crashed` state, as they are probably the ones that require our attention.

The tab in the middle that shows '1' is the `Completed` tab. It shows the flow runs that ended in `Completed` state.


Now, let's see how we can schedule this flow to run every day at 9:00 AM.

## Understanding Prefect Deployments

In the previous section, we have created a Prefect flow and ran it as a local process.
However, in real life, we would like to run our flows in a production environment, which is usually remote.
For example, we might want to run our flows in a Kubernetes cluster such as EKS, or in a serverless environment such as AWS ECS.
In order to run our flows in a production environment, we need to deploy them.

Prefect encapsulates the deployment logic in a Prefect entity called a Prefect Deployment.