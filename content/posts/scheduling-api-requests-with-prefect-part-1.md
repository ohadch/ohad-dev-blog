+++
title = 'Scheduling API Requests With Prefect - Part 1'
date = 2023-09-10T08:09:53+03:00
draft = false
+++

_Note: This post is part of a series of posts about Prefect._

## Introduction

One of the most common tasks in data engineering is to make API requests. 
This is a task that can be automated with Prefect. 
Prefect is an ecosystem for building, running, and monitoring data workflows. Its core is open source, and it is written in Python.

This is the first post in a series of posts about Prefect. At the end of this series, you will be able to create, run, and monitor Prefect flows.
In this post, we will learn about Prefect, and we will create a simple Prefect flow that makes a GET request to the Bored API, prints the response to the console, and saves it as a Prefect artifact.

You may view the source code for this post [here](https://github.com/ohadch/ohad-dev-blog-examples/tree/master/prefect-schedule-api-requests).

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

### An example DAG

Imagine you're getting ready for a cozy dinner at home with your family.

You have three main tasks: You need to buy groceries, prepare food, and prepare the table. Only then, you can eat.

So a simple DAG for this workflow could be:


```text
buy_groceries <- prepare_food <- prepare_table
```

_Notice that the arrow does not indicate the direction of the execution over time. 
Instead, it indicates the dependency between the tasks. 
For example, `prepare_food` depends on `buy_groceries`, and `prepare_table` depends on `prepare_food`._

That is a simple DAG. But it can be improved: while the food is being prepared, 
you can ask your significant other to prepare the table, so you can save some time by running these tasks in parallel.

So a better DAG for this workflow could be:

```text
buy_groceries <- prepare_food
buy_groceries <- prepare_table
```

In this DAG, `prepare_food` and `prepare_table` tasks depend on `buy_groceries` task.

This is a simple example, but it represents the general idea of DAGs.

This is how the three components of DAG stand for:

#### Directed

The edges in the graph have a direction. 
In our example, the direction is from `buy_groceries` to `prepare_food` and `prepare_table`.

#### Acyclic

There are no cycles in the graph. 
A cycle is a path that starts and ends at the same node, causing a mutual dependency between the nodes, 
leading to an infinite loop. 
In our example, there are no cycles in the graph. But if we add a dependency from `prepare_food` to `buy_groceries`,
we will have a cycle in the graph, as `buy_groceries` will depend on `prepare_food`, and `prepare_food` will depend on `buy_groceries`.

#### Graph 
A graph is a collection of nodes and edges. In our example, the nodes are `buy_groceries`, `prepare_food`, and `prepare_table`,
and the edges are the dependencies between the tasks.


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

```text
get_random_activity <- print_activity <- save_artifact
```

In this DAG, `print_activity` depends on `get_random_activity`, and `save_artifact` depends on `get_random_activity`.

We can even run `print_activity` and `save_artifact` in parallel, as they both depend on `get_random_activity`:
    
```text
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

## Prefect Tasks and Flows

The way to create a DAG in Prefect is to create a Prefect flow.

A Prefect flow is a Python function that is decorated with `@flow` decorator. It represents a DAG.
The tasks in the flow are represented by Python functions that are decorated with `@task` decorator.
Finally, the dependencies between the tasks are represented by the function calls in the flow function.

## The power of Prefect

It is important to note that literally any Python function can be a Prefect task. In my opinion, that is the main power of Prefect, compared to its alternatives: it allows you to create a DAG by simply writing a Python function. 
Moreover, it allows you to simply wrap your existing Python functions with `@task` decorator, and use them in your flow.
Not only that I find this approach very intuitive, I also find it very powerful.
It allowed me to easily integrate Prefect into my existing Python projects without having to change my code at all.

Let's see how we can create a Prefect flow for our use case.

## Creating a Prefect flow

Although we can create a Prefect project with the `prefect` CLI, for this example we will simply create the flow manually.

I like to create a `flows` directory for my flows and keep them there. So let's create a `flows` directory:

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
    activity = get_random_activity()
    
    # Then, we can both print it to the console, and save it as an artifact, but we can do it in parallel
    print_activity(activity)
    save_activity_artifact(activity)
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
    activity = get_random_activity()
    print_activity(activity)
    save_activity_artifact(activity)


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

We may also visualize the flow. In order to do so, simply edit the `if __name__ == "__main__"` block at the end of the file to look like this:

```python
if __name__ == "__main__":
    suggest_activity.visualize()
```

Then, run the flow again:

```bash
python flows/suggest_activity.py
```

A preview of the flow should pop up:

![Flow preview](/posts/scheduling-api-requests-with-prefect-part-1/flow-visualization.png)

As you can see, the flow is represented as a DAG. The nodes in the graph represent the tasks, and the edges represent the dependencies between the tasks.
Just like in our example, the `print_activity` and `save_activity_artifact` tasks are represented as nodes, and they both depend on the `get_random_activity` task.

## Conclusion for this post

In this post, we:
- Learned about Prefect
- Got familiar with DAGs
- Implemented a simple Prefect flow that makes a GET request to the Bored API, prints the response to the console, and saves it as a Prefect artifact.
- Learned how to visualize a Prefect flow

In the [next post](/posts/scheduling-api-requests-with-prefect-part-2/), we will learn how to schedule our flow to run every day at 9:00 AM.
