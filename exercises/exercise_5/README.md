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

4. Use the `docker app inspect` command and set the `vote.enabled` parameter to false. You'll see that the `vote` service is not included, as well as the `vote.enabled` parameter is set to false.

    <details>
      <summary>Sample Output</summary>

    ```console
    $ docker app inspect -s vote.enabled=false
    voting-app 0.1.0

    Maintained by: root

    Services (4) Replicas Ports Image
    ------------ -------- ----- -----
    results      1        5001  mikesir87/examplevotingapp_result
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
    vote.enabled    false
    ```
    </details>


## Using our Application Stack for local development

While doing local development, we don't necessarily need to run multiple replicas of any service. We also may not have a full swarm locally to deploy everything, which could cause problems (for example, one a single-node swarm, some placement constraints may fail if you're trying to place things only on a worker node).

Remember that `docker app render` can render a compose file. The Docker Compose command can read a compose file from stdin, so we can pipe the two commands together.

1. In your dev instance, spin up the application stack using Docker Compose. Remember to disable the `vote` service when rendering the app.

    ```console
    $ docker app render -s vote.enabled=false | docker-compose -f - up -d
    ```

    The `-f -` flag in the compose command uses the compose file coming from stdin. You should see all but the `vote` service startup.

2. Clone the voting app repo (found at github.com/mikesir87/example-voting-app).

    ```console
    $ git clone https://github.com/mikesir87/example-voting-app.git
    ```

3. In the repo's directory, start _only_ the `vote` service in the `docker-compose.yml` file. You should see it start up and the port badge for `5000` appear. Once it appears, click on it to validate the `vote` service has started.

    ```console
    $ docker-compose up -d vote
    Compose does not use swarm mode to deploy services to multiple nodes in a swarm. All containers will be scheduled on the current node.

    To deploy your application across the swarm, use `docker stack deploy`.

    Creating example-voting-app_vote_1 ... done
    ```

4. Now, let's make a change to the application! In the `vote/templates/index.html` file, let's make a change. You can change anything, but one idea might be to adjust the "Tip: you can change your vote" text to say "Tip: try changing your vote!". After saving the change, refresh the page displaying the vote app and you should see the change.

5. Now that we've made the change we want to save in our app, let's stop the dev container.

    ```console
    $ docker-compose down
    Stopping example-voting-app_vote_1 ... done
    Removing example-voting-app_vote_1 ... done
    ```

6. Let's build our new image and push it to Docker Hub. Tag the image using `[your-docker-hub-username]/updated-vote-app`.

    ```console
    $ cd vote
    $ docker build -t mikesir87/updated-vote-app .
    ```

6. Back in our docker app, let's update the `vote` service to use our new image.

    ```yaml
    services:
      vote:
        image: mikesir87/updated-vote-app
    ```

7. Push your updated app and then deploy it to your cluster. Validate that your vote service has the updated image!

