+++
title = 'Scheduling API Requests With Prefect - Part 2'
date = 2023-09-10T08:09:53+03:00
draft = true
+++

## Introduction

In the [previous post](/posts/scheduling-api-requests-with-prefect-part-1/), we have learned about Prefect,
and we have created a simple Prefect flow that makes a GET request to the Bored API, prints the response to the console, and saves it as a Prefect artifact.

In this post, we will learn how to schedule our flow to run every day at 9:00 AM.

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
![Prefect UI](/posts/scheduling-api-requests-with-prefect-part-2/prefect-dashboard.png)


This is the Prefect UI. Currently, we see the dashboard page, 

Inside the `Flow Runs` box, we see flow runs broken down by state. 
By default, we see the flow runs that are in `Crashed` state, as they are probably the ones that require our attention.

The tab in the middle that shows '1' is the `Completed` tab. It shows the flow runs that ended in `Completed` state.

## Understanding Prefect Architecture

In the previous post, we have created a Prefect flow and ran it as a local process.
Although it works, it is not a good solution for scheduling our flow to run every day at 9:00 AM.

In order to schedule our flow to run every day at 9:00 AM, we need to deploy it as a Prefect deployment,
that can be scheduled to be pulled by a Prefect worker and executed on an environment of our choice.

Let's dive into Prefect architecture to understand how this works.

![Prefect Architecture](/posts/scheduling-api-requests-with-prefect-part-2/prefect-architecture.png)

As you can see in the image above, Prefect consists of following components:
- Prefect API: The Prefect API is a REST API that is used to manage flows, flow runs, and deployments.
- Prefect UI: The Prefect UI is a web application that is used to manage flows, flow runs, deployments and more.
- Work pools: A work pool is a kind of a queue that is used to store flow runs that are ready to be executed.
- Prefect worker: The Prefect worker is a Python server that lives on the execution environment. It is responsible for pulling flow runs from the work pool, and executing them.

So, in order to schedule our flow to run every day at 9:00 AM, we need to deploy it as a Prefect deployment,
and then schedule the deployment to be pulled by a Prefect worker every day at 9:00 AM.

Let's see how we can do that.

## Deploying a Prefect flow

TBD...