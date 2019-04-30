# Exercise - Creating the Docker Application

> **Time**: Approximately 20 minutes
>
> **Difficulty**: Easy

## Exercise Objectives

By the end of this exercise, you will have:

- Created a Docker Application package
- Learned about the application package components
- Learned about merging and splitting the components
- Learned how to inspect and validate the application



## Docker Application Overview

Application packages are a construction above compose files to improve application lifecycle and workflow, from development to test to production. An application package is a set of 3 documents:

1. A `metadata.yml` file describing the application metadata (name, version, description, ...)
2. A `docker-compose.yml` file describing the application structure (what we have right now)
3. A `parameters.yml` file with key/value parameters (we will use this in the next exercise)

You can also include any of your own custom files, including config files. These additional files are called `attachments`.

![Application](application.png)

## Initialize the Application

1. Let's look at the `docker app init` command and see it's options.

    ```bash
    $ docker app init --help

    Usage:  docker app init APP_NAME [--compose-file COMPOSE_FILE] [--description DESCRIPTION] [--maintainer NAME:EMAIL ...] [OPTIONS]

    Start building a Docker Application package. If there is a docker-compose.yml file in the current directory it will be copied and used.

    Examples:
    $ docker app init myapp --description "a useful description"

    Options:
          --compose-file string      Compose file to use as application base (optional)
          --description string       Human readable description of your application (optional)
          --maintainer stringArray   Name and email address of person responsible for the application
                                    (name:email) (optional)
          --single-file              Create a single-file Docker Application definition
    ```

2. Let's now create our Docker app! We will use the `docker app init` command and specify the description and maintainer. Feel free to change these values. Be sure to run this in the same directory as the `docker-compose.yml` file we were working on in the last exercise. Otherwise you can specify a path to your compose file using `--compose-file path/to/my/docker-compose.yml`.

    ```bash
    $ docker app init voting-app --description "Voting App" --maintainer "dapworkshop:dapworkshop@docker.com"
    Created "voting-app.dockerapp"
    ```

3. If you run `tree`, you'll see a new directory. The name is your app name with a `.dockerapp` suffix. Let's look inside the directory.

    ```bash
    $ tree
    .
    ├── docker-compose.yml
    └── voting-app.dockerapp
        ├── docker-compose.yml
        ├── metadata.yml
        └── parameters.yml

    1 directory, 4 files
    ```

4. The compose file is a copy of the file you were working with earlier. If you open the `metadata.yml` file, you'll see the config we specified during initialization.

    ```bash
    $ cat voting-app.dockerapp/metadata.yml
    # Version of the application
    version: 0.1.0
    # Name of the application
    name: voting-app
    # A short description of the application
    description: Voting App
    # List of application maintainers with name and email for each
    maintainers:
      - name: dapworkshop
        email: dapworkshop@docker.com
    ```



## Merging our Docker Application

While the default app initialization is to create a new folder and create three separate files, you can actually merge everything into a single file!

1. Use the `docker app merge` command to merge the app into a single file. We use the `-o` flag to specify the new file to create.

    ```bash
    $ docker app merge voting-app -o voting-app-merged.dockerapp
    ```

2. If you look at the file, you'll see the contents of each of the original files merged together. For brevity, the full contents aren't included below.

    <details>
      <summary>Full output of merged file</summary>

    ```bash
    $ cat voting-app-merged.dockerapp
    # Version of the application
    version: 0.1.0
    # Name of the application
    name: voting-app
    # A short description of the application
    description:
    # List of application maintainers with name and email for each
    maintainers:
    - name: root
        email:

    ---
    version: "3.7"

    services:
    vote:
        image: mikesir87/examplevotingapp_vote
        networks:
        - frontend
        ports:
        - 5000:80
        depends_on:
        - redis
        deploy:
        replicas: 2
        update_config:
            parallelism: 2
        restart_policy:
            condition: on-failure

    redis:
        image: redis:alpine
        networks:
        - frontend
        deploy:
        replicas: 1
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
        replicas: 1
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
        volumes:
        - db-data:/var/lib/postgresql/data
        deploy:
        placement:
            constraints: [node.role == manager]

    results:
        image: mikesir87/examplevotingapp_result
        ports:
        - 5001:80
        networks:
        - backend
        depends_on:
        - db
        deploy:
        replicas: 1
        update_config:
            parallelism: 2
            delay: 10s
        restart_policy:
            condition: on-failure

    networks:
    frontend:
        name: front-tier
    backend:
        name: back-tier

    volumes:
    db-data:
    ---
    {}
    ```

## Splitting the Docker Application

Once merged, the application can be split back into separate files using the `docker app split` command.

1. Run the `docker app split` command to split the file back out into separate files

    ```bash
    $ docker app split voting-app-merged.dockerapp -o voting-app-split.dockerapp
    ```

2. Look in the `voting-app-split.dockerapp` directory and you'll see the files split apart.

    ```bash
    $ ls voting-app-merged.dockerapp
    docker-compose.yml  metadata.yml        parameters.yml
    ```

3. At this point, we don't need the split and merged versions anymore. Let's remove them to reduce confusion going forward.

    ```bash
    $ rm -r voting-app-merged.dockerapp voting-app-split.dockerapp
    ```


## Inspecting our Docker Application

We can use the `docker app inspect` command to get a quick output of all of the services, number of replicas, ports, and the image being used. This could allow an ops admin to quickly check all the elements before deploying to production, without having to manually parse the compose file.

1. Run the `docker app inspect` command to inspect our application. You shouldn't need to provide the app name, as there is only one in the directory.

    ```bash
    $ docker app inspect
    voting-app 0.1.0

    Maintained by: dapworkshop <dapworkshop@docker.com>

    Voting App

    Services (6) Replicas Ports   Image
    ------------ -------- -----   -----
    vote         2        5000    mikesir87/examplevotingapp_vote
    redis        1                redis:alpine
    worker       1                dockersamples/examplevotingapp_worker
    db           1                postgres:9.4
    results      1        5001    mikesir87/examplevotingapp_result

    Networks (3)
    ------------
    frontend
    backend
    proxy

    Volume (1)
    ----------
    db-data
    ```



## Validating our Application

Before we're ready to ship our application, we should validate it to make sure everything is set. Specifically, validation does the following:

- Ensures our compose file is valid (correct syntax, etc.)
- Ensures required metadata is provided (name, version) and is the correct format
- Ensures all parameters (which we'll talk about next) have default values

1. Run the `docker app validate` command to make sure our application is valid.

    ```bash
    $ docker app validate
    Validated "/root/voting-app.dockerapp"
    ```

2. Let's make a change to invalidate the application. In the `voting-app.dockerapp/metadata.yml` file, comment out the `version` field using a `#`. Then, revalidate.

    ```bash
    $ docker app validate
    failed to validate metadata:
    - version: version is required
    ```

    We have an error! Go ahead and fix it and revalidate.
