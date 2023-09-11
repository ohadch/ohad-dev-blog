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

## Understanding DAGs

In order to understand Prefect, we need to understand DAGs. DAG stands for Directed Acyclic Graph. 
In the context of workflows, a DAG is a graph that represents the dependencies between tasks. 
Each node in the graph represents a task, and each edge represents a dependency between tasks.

For example, let's say we have a workflow that consists of three tasks: `task1`, `task2`, and `task3`.
`task1` depends on `task2`, and `task2` depends on `task3`.

Possible DAG for this workflow could be:

```
task1 <- task2 <- task3
```

In this DAG, `task3` depends on `task2`, and `task2` depends on `task1`.

This is how the the three components of DAG stand for:
- Directed: The edges in the graph have a direction. In our example, the direction is from `task1` to `task2`, and from `task2` to `task3`.
- Acyclic: There are no cycles in the graph. Aka, there are no loops in the graph. In our example, if `task1` depends on `task3`, then we have a cycle in the graph.
- Graph: The graph is a collection of nodes and edges. In our example, the nodes are `task1`, `task2`, and `task3`. The edges are the dependencies between the tasks.

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

Now, let's say we want to get a random activity every day at 9:00 AM, and we want to print it to the console. 
I know, it is not the most exciting use case, but it is a good example for our purposes. 

A sample DAG for this workflow could be as simple as:

```
get_random_activity -> send_email
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
    print(f"Today's activity is: {activity}")
```

Finally, let's create a `suggest_activity` flow. This flow will call the `get_random_activity` task, and then call the `print_activity` task:

```python
@flow
def suggest_activity() -> None:
    """
    Suggest a random activity
    """
    activity = get_random_activity.submit()
    print_activity.submit(activity)
```

That's it! We have created our flow. This is how the `flows/suggest_activity.py` file should look like:

```python
from prefect import flow, task


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
    print(f"Today's activity is: {activity}")


@flow
def suggest_activity() -> None:
    """
    Suggest a random activity
    """
    activity = get_random_activity.submit()
    print_activity.submit(activity)


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
22:49:42.857 | INFO    | prefect.engine - Created flow run 'upbeat-pogona' for flow 'suggest-activity'
22:49:43.148 | INFO    | Flow run 'upbeat-pogona' - Created task run 'get_random_activity-0' for task 'get_random_activity'
22:49:43.149 | INFO    | Flow run 'upbeat-pogona' - Submitted task run 'get_random_activity-0' for execution.
22:49:43.173 | INFO    | Flow run 'upbeat-pogona' - Created task run 'print_activity-0' for task 'print_activity'
22:49:43.174 | INFO    | Flow run 'upbeat-pogona' - Submitted task run 'print_activity-0' for execution.
22:49:44.282 | INFO    | Task run 'get_random_activity-0' - Finished in state Completed()
Today's activity is: Make a simple musical instrument
22:49:44.358 | INFO    | Task run 'print_activity-0' - Finished in state Completed()
22:49:44.386 | INFO    | Flow run 'upbeat-pogona' - Finished in state Completed('All states completed.')
```

As you can see, the flow ran successfully, and we got a random activity printed to the console.

Now, let's see how we can schedule this flow to run every day at 9:00 AM.





