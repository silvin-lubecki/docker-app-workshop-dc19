# Exercise 4 - Configure your Application with Parameters

> **Time**: Approximately 15 minutes
>
> **Difficulty**: Easy

## Table of Contents

1. [Docker App Parameters](#docker-app-parameters)
1. [Using a Parameter to Change Voting Options](#using-a-parameter-to-change-voting-options)
1. [Upgrading a Deployed Application](#upgrading-a-deployed-application)
1. [More practice: Prepare for production](#more-practice-prepare-for-production)

## Exercise Objectives

By the end of this exercise, you will have:

- Learned about Docker App parameters
- Added parameters to the voting app application
- Learned how to deploy an update to an already-deployed application



## Docker App Parameters

In addition to using environment variables, Docker App allows you to define parameters for an application that can be overridden at deploy-time. Examples of parameters might include port mappings, cpu/memory limits, number of replicas, label definitions, or network names. _Almost_ everything in the Docker App compose file can be parameterized, **except for the image names**.


## Using a Parameter to Change Voting Options

By default, the vote and results services let you vote between Dogs and Cats. However, wouldn't it be neat if we could change that? Conveniently, both the `vote` and `results` services allow the definition of two environment variables (`OPTION_A` and `OPTION_B`) to let us change the voting choices. This sounds like a great opportunity for parameters!

1. In both the `vote` and `results` services, add environment variables named `OPTION_A` and `OPTION_B` with a value of `${options.A}` and `${options.B}`. The values are simply placeholders for the parameters.

    <details>
      <summary>Solution</summary>
    
    ```yaml
    services:
      vote:
        environment:
          OPTION_A: ${options.A}
          OPTION_B: ${options.B}
      results:
        environment:
          OPTION_A: ${options.A}
          OPTION_B: ${options.B}
    ```
    </details>

2. Before setting the parameter, run `docker app build voting-app -t <your-hub-username>/voting-app.dockerapp && docker app inspect <your-hub-username>/voting-app.dockerapp --pretty` to see what happens.

    <details>
      <summary>Full Output</summary>
    
    ```console
    $ docker app build voting-app -t <your-hub-username>/voting-app.dockerapp && docker app inspect <your-hub-username>/voting-app.dockerapp
    … # Build log
    inspect failed: Action "com.docker.app.inspect" failed: failed to load Compose file: invalid interpolation format for services.vote.environment.OPTIONS_A: "required variable options.A is missing a value". You may need to escape any $ with another $.
    ```
    </details>

    What happened? The command fails because "required variable options.A is missing a value." In other words, **all parameters are required to have default values.** Those default values are specified in the `parameters.yml` file.

3. Specify the default values for `options.A` and `options.B` in the `parameters.yml` file. Let's set `options.A` to "Cats" and `options.B` to "Dogs". Remember that this file is a YAML file.

    <details>
      <summary>Solution</summary>
    
    ```yaml
    options:
      A: Cats
      B: Dogs
    ```
    </details>

4. Now, re-run the `docker app inspect --pretty` command to look at the output. Do not forget to rebuild the application image. You should see the parameters with their default values in the output now!

    <details>
      <summary>Full output</summary>
    
    ```console
    $ docker app build voting-app -t <your-hub-username>/voting-app.dockerapp
    … 
    $ docker app inspect --pretty
    voting-app 0.1.0

    Maintained by: root

    Services (5) Replicas Ports Image
    ------------ -------- ----- -----
    db           1              postgres:9.4
    worker       1              dockersamples/examplevotingapp_worker
    result       1        5001  mikesir87/examplevotingapp_result
    vote         2        5000  mikesir87/examplevotingapp_vote
    redis        1              redis:alpine

    Networks (2)
    ------------
    backend
    frontend

    Volume (1)
    ----------
    db-data

    Parameters (2) Value
    -------------- -----
    options.A      Cats
    options.B      Dogs
    ```
    </details>

5. At this point, go ahead and deploy the Docker App

    <details>
      <summary>Full output</summary>
    
    ```console
    $ docker app run voting-app --name voting-app --target-context=swarm
    Creating network back-tier
    Creating network front-tier
    Creating service voting-app_redis
    Creating service voting-app_db
    Creating service voting-app_worker
    Creating service voting-app_results
    Creating service voting-app_vote
    Application "voting-app" installed on context "swarm"
    ```
    </details>

    Once it's deployed, go ahead and check out the vote and results apps. You should see that we're now using Cats vs Dogs!


## Upgrading a Deployed Application

With the app deployed, let's change the settings by "upgrading" the application bundle. To do so, we can use the `docker app upgrade` command. While we will change settings in the upgrade here, you can use this command to actually deploy an updated version of the app.

Suddenly the project manager and the design team changed their minds. Cats and Dogs are so 2018, voting options must be changed to Moby and Molly. Add the `-s` flag to upgrade command to set options.A to "Moby" and optionB to "Molly".

<details>
  <summary>Solution/Output</summary>

```console
$ docker app upgrade voting-app -s options.A=Moby -s options.B=Molly --target-context=swarm
Updating service voting-app_results (id: tpugiytt4eq9p88lvb8900pmq)
Updating service voting-app_vote (id: d49hxltgvg5faie0kc735oy42)
Updating service voting-app_redis (id: x9hpof20yumf2gv3mbbd9g1i5)
Updating service voting-app_db (id: nwssvpk4r8gklcfnvd47w7tzx)
Updating service voting-app_worker (id: qoyl03yaxtdyefb5oh6u9m698)
Application "voting-app" upgraded on context "swarm"
```
</details>
<br/>

After a moment, you should be able to open either service and see the options have been updated. Welcome to 2019! :tada:


## More Practice: Prepare for production

While we added parameters to change the options, we can set parameters for almost anything in the compose file.

As a practice:
1. Turn the exposed ports into parameters (`vote.exposedPort` and `results.exposedPort`) and allow them to be overridden.
1. Add a parameter to all the `replicas` sections, don't forget the default value
1. Add a `resources` section under `deploy`:

```yaml
services:
  service:
    deploy:
      resources:
        limits:
          cpus: '${service.cpu_limit}'
          memory: ${service.memory_limit}
```

<details>
  <summary>docker-compose.yml</summary>

```yaml
version: "3.7"

services:
  vote:
    image: mikesir87/examplevotingapp_vote
    networks:
      - frontend
    depends_on:
      - redis
    ports:
      - ${vote.exposedPort}:80
    deploy:
      replicas: ${vote.replicas}
      resources:
        limits:
          cpus: '${vote.cpu_limit}'
          memory: ${vote.memory_limit}
      update_config:
        parallelism: 2
      restart_policy:
        condition: on-failure
    environment:
      OPTION_A: ${options.A}
      OPTION_B: ${options.B}

  redis:
    image: redis:alpine
    networks:
      - frontend
    deploy:
      replicas: ${redis.replicas}
      resources:
        limits:
          cpus: '${redis.cpu_limit}'
          memory: ${redis.memory_limit}
      update_config:
        parallelism: 2
        delay: 10s
      restart_policy:
        condition: on-failure

  worker:
    image: dockersamples/examplevotingapp_worker
    networks:
      - frontend
      - backend
    deploy:
      replicas: ${worker.replicas}
      resources:
        limits:
          cpus: '${worker.cpu_limit}'
          memory: ${worker.memory_limit}
      restart_policy:
        condition: on-failure
        delay: 10s
        max_attempts: 3
        window: 120s
      placement:
        constraints: [node.role == manager]

  db:
    image: postgres:9.4
    networks:
      - backend
    deploy:
      placement:
        constraints: [node.role == manager]

  results:
    image: mikesir87/examplevotingapp_result
    networks:
      - backend
    depends_on:
      - db
    ports:
      - target: 80
        published: ${results.exposedPort}
        protocol: tcp
        mode: host
    deploy:
      replicas: ${results.replicas}
      resources:
        limits:
          cpus: '${results.cpu_limit}'
          memory: ${results.memory_limit}
      update_config:
        parallelism: 2
        delay: 10s
      restart_policy:
        condition: on-failure
    environment:
      OPTION_A: ${options.A}
      OPTION_B: ${options.B}

networks:
  frontend:
    name: front-tier
  backend:
    name: back-tier
```
</details>

<details>
  <summary>parameters.yml</summary>

```yaml
options:
  A: Cats
  B: Dogs
vote:
  replicas: 2
  cpu_limit: 1
  memory_limit: 512M
  exposedPort: 5000
redis:
  replicas: 1
  cpu_limit: 1
  memory_limit: 512M
worker:
  replicas: 1
  cpu_limit: 1
  memory_limit: 512M
results:
  replicas: 1
  cpu_limit: 1
  memory_limit: 512M
  exposedPort: 5001
```
</details>
<br/>

Now we will create a new parameters file for **production**, with different parameters:
- more replicas
- more realistic constraints on cpu and memory limits

:warning: Be sure you uninstall the previous installation before installing this one for production.

Then we are ready for installation:
```sh
$ docker app run voting-app --parameters-file=production.yml --target-context=swarm
```
