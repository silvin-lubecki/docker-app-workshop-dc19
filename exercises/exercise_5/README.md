# Exercise 5 - Selectively Disabling Services

> **Time**: Approximately 15 minutes
>
> **Difficulty**: Medium

## Table of Contents

1. [Why disable services?](#why-disable-services)
2. [How to disable services](#how-to-disable-services)
3. [Updating the Vote and Results services](#updating-the-vote-and-results-services)
4. [Using our Application Stack for local development](#using-our-application-stack-for-local-development)


## Exercise Objectives

By the end of this exercise, you will have:

- Learned about a few use cases for disabling of services
- Updated the vote and results services to allow them to be disabled
- Deployed the application with the vote app disabled and started a dev-enabled container for the vote app that hooks into the application stack

## Why disable services?

In most deployments, you may not want to disable a service. However, there are a few valid use cases:

- Disabling a database service because you want to connect to a managed database service (like Amazon's RDS)
- Disabling a web application when deploying locally because you want to make changes to one of the services (run that service in a dev-enabled container)
- Disabling a proxy service because you already have a proxy in your cluster (yes, we don't have a reverse proxy, but you can add one in [Exercise 7](../exercise_7/README.md))

For this exercise, we are going to instrument only the `vote` and `results` services for disabling.


## How to disable services

As we learned in the previous exercise, we can add parameters to modify our compose file and allow overrides at deploy time. Since the compose file has to be rendered, the capability to disable a service was added.

Starting with compose schema version 3.4, any compose file can include extension fields, which all start with `x-`. If a service contains a `x-enabled` field, Docker App will only include that service if it's value is "truthy." For example...

```yaml
services:
  app:
    x-enabled: false
    image: nginx
```

In this case, the `app` service will _always_ be disabled, as the value is false. However, we probably don't want to _always_ disable a service (why include it in the first place?). Therefore, it sounds like a great candidate for a parameter!


## Updating the Vote and Results services

To update the vote and results services, let's do the following:

1. In the `vote` service, add a `x-enabled` field that has a value of `${vote.enabled}`.

    <details>
      <summary>Solution</summary>

    ```yaml
    services:
      vote:
        x-enabled: ${vote.enabled}
    ```
    </details>

2. Update the `parameters.yml` to set a default value for `vote.enabled` to `true`.

    <details>
      <summary>Solution</summary>

    ```yaml
    vote:
      enabled: true
    ```
    </details>

3. Do the same thing for the `results` service, but use `results.enabled` as the parameter name.

4. Do not forget to build the application image.

```sh
$ docker app build voting-app -t <your-hub-username>/voting-app.dockerapp
```

5. Use the `docker app inspect` command. You'll see the `vote.enabled` and `results.enabled` parameters are set to the default value `true`.

    <details>
      <summary>Sample Output</summary>

    ```console
    $ docker app inspect <your-hub-username>/voting-app.dockerapp --pretty
    voting-app 0.1.0

    Maintained by: root

    Services (4) Replicas Ports Image
    ------------ -------- ----- -----
    results      1        5001  mikesir87/examplevotingapp_result
    vote         2        5000  mikesir87/examplevotingapp_vote
    redis        1              redis:alpine
    db           1              postgres:9.4
    worker       1              dockersamples/examplevotingapp_worker

    Networks (2)
    ------------
    backend
    frontend

    Volume (1)
    ----------
    db-data

    Parameters (4)  Value
    --------------  -----
    optionA         Cats
    optionB         Dogs
    results.enabled true
    vote.enabled    true
    ```
    </details>

## Using our Application Stack for development

In this section we will see how we can use the `x-enabled` flag to develop and test a service of our application.

1. First of all we need to add a `attachable: true` to the frontend network in the `docker-compose.yml`. This option is required to allow a standalone container to attach to an overlay network created by a stack.

    <details>
      <summary>Solution</summary>

    ```yaml
    networks:
      frontend:
        name: front-tier
        attachable: true
    ```
    </details>    


2. Deploy the application without enabling the `vote` service. Ensure with the `docker service ls` that the vote service has not been deployed

    <details>
      <summary>Solution</summary>

    ```console
    $ docker app run <your-hub-username>/voting-app.dockerapp --name voting-app -s vote.enabled=false  --target-context=swarm
    Creating network front-tier
    Creating network back-tier
    Creating service voting-app_db
    Creating service voting-app_results
    Creating service voting-app_redis
    Creating service voting-app_worker

    $ docker service ls
    ID                  NAME                 MODE                REPLICAS            IMAGE                                          PORTS
    89zwowq45acp        voting-app_db        replicated          1/1                 postgres:9.4                                   
    nza74rbbrwde        voting-app_redis     replicated          1/1                 redis:alpine                                   
    dlhpp09kfhee        voting-app_results   replicated          1/1                 mikesir87/examplevotingapp_result:latest       
    j24jm0lyp8de        voting-app_worker    replicated          1/1                 dockersamples/examplevotingapp_worker:latest   
    ```
    </details>

3. Deploy the `vote` service as a standalone container, attached to the `frontend` network with `OPTION_A` and `OPTION_B` environment variable set to `Cats` and `Dogs`. You should see the port badge for `5000` appear on the master node. Once it appears, click on it to validate the `vote` service has started.

    <details>
      <summary>Solution</summary>

    ```console
    $ docker --context swarm container run --name voting-app-vote-svc-dev -d --rm -e OPTION_A=Cats -e OPTION_B:Dogs -p 5000:80 --network front-tier mikesir87/examplevotingapp_vote
    ```
    <details>

4. Clone the voting app repo (found at github.com/mikesir87/example-voting-app).

    ```console
    $ git clone https://github.com/mikesir87/example-voting-app.git
    ```

5. Now, let's make a change to the application! In the `vote/templates/index.html` file, let's make a change. You can change anything, but one idea might be to adjust the "Tip: you can change your vote" text to say "Tip: try changing your vote!". After saving the change, refresh the page displaying the vote app and you should see the change.

6. Now that we've made the change we want to save in our app, let's stop the dev container.

    ```console
    $ docker container stop $(docker container ls --filter name=voting-app-vote-svc-dev -q)
    ```

7. Let's build our new image and push it to Docker Hub. Tag the image using `[your-docker-hub-username]/updated-vote-app`.

    ```console
    $ cd vote
    $ docker build -t <your-docker-hub-username>/updated-vote-app .
    …
    $ docker push <your-docker-hub-username>/updated-vote-app
    ```

8. Back in our docker app, let's update the `vote` service to use our new image.

    ```yaml
    services:
      vote:
        image: <your-docker-hub-username>/updated-vote-app
    ```

9. Build your updated app and then deploy it to your cluster. Validate that your vote service has the updated image!

