+++
title = 'Scheduling API Requests With Prefect - Part 2'
date = 2023-09-10T08:09:53+03:00
draft = true
+++

## Introduction

In the [previous post](/posts/scheduling-api-requests-with-prefect-part-1/), we have learned about Prefect,
and we have created a simple Prefect flow that makes a GET request to the Bored API, prints the response to the console, and saves it as a Prefect artifact.
Nevertheless, we ran our flow directly from the CLI as a Python script.
Although it works, it is not a good solution for scheduling our flow in production. 

In this post, we will learn how to schedule our flow to run every day at 9:00 AM.
By the end of this post, we will have a Prefect flow that makes a GET request to the Bored API every day at 9:00 AM, and saves the response as a Prefect artifact.

## Assumptions

As in the previous post, we will assume that we have a Python 3.10+ environment with Prefect installed.
Once again, we will only cover the basics of each topic. 
For more details, please refer to the [Prefect documentation](https://docs.prefect.io/).

## A brief view of the Prefect architecture

Before we dive into the details, we need to have a basic understanding of the Prefect architecture.

![Prefect Architecture](/posts/scheduling-api-requests-with-prefect-part-2/prefect-architecture.png)

As we can see in the image above, Prefect consists of following components:
- **Prefect Server**: Python server that is responsible for managing the Prefect API, registering flows, and storing flow runs and artifacts.
- **Prefect UI**: Web application client for Prefect Server.
- **Work pools**: Like a pub/sub topic that is used to distribute work to Prefect workers.
- **Prefect worker**: Python server that lives on the execution environment. It is responsible for pulling flow runs from the work pool, and executing them.

_Of course, this is a simplified view of the Prefect architecture. For more details, please refer to the [Prefect documentation](https://docs.prefect.io/)._

## An overview of the Process

In a nutshell, we need to deploy our flow as a Prefect deployment, 
and then schedule the deployment to be pulled by a Prefect worker every day at 9:00 AM.

It all comes together like this:
1. We create a Prefect flow (that's what we did in the [previous post](/posts/scheduling-api-requests-with-prefect-part-1/)).
2. We deploy the flow as a **Prefect Deployment**: a versioned and executable representation of a flow. 
   A deployment is the entity that connects the flow code to the execution environment.
3. We schedule the deployment to be pulled by a Prefect worker every day at 9:00 AM.
4. The Prefect worker pulls the deployment from the work pool, and executes it on the execution environment where it lives.
5. Upon completion (or failure) of the flow run, the Prefect worker informs the Prefect API about the result.

## Deploying our flow

Prefect allows us to deploy our flow using a YAML file, or using the Python API. In this post, we will deploy our flow using the Python API.

### Creating a deployment

Let's add a new `deploy.py` file to the root of our project. Then, let's add the following code to it:

```python
import asyncio
from prefect.deployments import Deployment
from flows.suggest_activity import suggest_activity


async def main():
    # This line creates a deployment for the flow, which will be scheduled to run every day at 9:00 AM
    deployment = await Deployment.build_from_flow(
        name="Suggest Activity - Daily",
        flow=suggest_activity,
        cron="0 9 * * *",
    )

    await deployment.apply()


if __name__ == '__main__':
    asyncio.run(main())
```

As we can see, we are importing our flow from the `flows.suggest_activity` module, 
as well as the `Deployment` class from the `prefect.deployments` module and the `asyncio` module.

The reason for importing the `asyncio` module is that the Prefect API is asynchronous,
and in order to use it, we need to run our code in an `async` function. That's why we are using the `asyncio.run()` method to run our `main()` function.
If you are not familiar with the `asyncio` module, I recommend reading [this article](https://realpython.com/async-io-python/).

In the `main()` function, we are creating a new deployment for our flow, and we are scheduling it to run every day at 9:00 AM.
The `cron` argument is a cron expression that defines the schedule of the deployment.
In our case, we are using the following cron expression: `0 9 * * *`. 
If you are not familiar with cron expressions, I suggest visiting [crontab.guru](https://crontab.guru/) and playing around with it.

Finally, we are applying the deployment to the Prefect API by calling the `apply()` method.

That's all the code we need to create a deployment for our flow. 
Now, let's learn how to spin up the environment and run the deployment.

### Launching the Prefect environment

This is the place where understanding the Prefect architecture comes in handy.

In the previous section, we used our Python code to request with the Prefect API to create a deployment for our flow.


## The Prefect UI

Now it is a good time to get familiar with the Prefect UI. In order to do that, let's start the Prefect server:

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
![Prefect UI](/posts/scheduling-api-requests-with-prefect-part-2/prefect-dashboard.png)

This is the Prefect UI. Currently, we see the dashboard page, that gives us a high level overview of our flow runs and work pools. 
In the sidebar, we can see the different pages that we can navigate to. 
In this post, we will discuss the `Flow Runs`, `Deployments` and `Artifacts` pages.

So, in order to schedule our flow to run every day at 9:00 AM, we need to deploy it as a Prefect deployment,
and then schedule the deployment to be pulled by a Prefect worker every day at 9:00 AM.

Let's see how we can do that.

## Deploying a Prefect flow

TBD...