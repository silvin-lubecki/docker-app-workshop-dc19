# Exercise 8 - Advanced - Create a CNAB bundle from scratch

> **Time**: Approximately 15 minutes
>
> **Difficulty**: Medium

## Table of Contents
1. [Create a CNAB Bundle from scratch](#create-a-cnab-from-scratch)
1. [Use Docker App as a standard CNAB runtime](#use-docker-app-as-a-standard-cnab-runtime)

## Exercise Objectives

By the end of this exercise you will:

- Learn how to create a CNAB from scratch
- Learn how to use Docker App as a CNAB client

## Create a CNAB Bundle from scratch

Let's put our hands on what we just saw in the [Exercise 6](../exercise_6/README.md) and create our first CNAB!

1. Create a new directory called `simple`
1. In it, copy/paste the previous bundle example in a new file named `bundle.json`
1. Still in `simple`, create the following hierarchy:  

    ```bash
    $ tree simple
    simple
    ├── bundle.json
    └── cnab
        ├── app
        │   └── run
        └── build
            └── Dockerfile

    3 directories, 3 files
    ```

### Creating the Dockerfile for the invocation image

<details>
    <summary>Solution</summary>

```Dockerfile
FROM alpine:3.9.3
COPY cnab/app/run /cnab/app/run
RUN chmod +x /cnab/app/run
CMD /cnab/app/run
```
</details>
<br/>

### Creating the CNAB run executable
- Read the environment variable `CNAB_ACTION` and print a specific action result on the standard output. It must also fail if the action is unknown. According to our `bundle.json`, it must understand `install`/`upgrade`/`uninstall`/`io.cnab.status` actions.
- For each action, print the installation name.
- The `install` action must print the bundle name and the bundle version
- The `io.cnab.status` action must print the current port parameter

<details>
    <summary>Solution</summary>

```bash
#!/bin/sh

action=$CNAB_ACTION
name=$CNAB_INSTALLATION_NAME

case $action in
    install)
    echo "Install action"
    echo "Bundle $CNAB_BUNDLE_NAME version $CNAB_BUNDLE_VERSION"
    ;;
    uninstall)
    echo "Uninstall action"
    ;;
    upgrade)
    echo "Upgrade action"
    ;;
    io.cnab.status)
    echo "Status action"
    echo "current simple_component_port $SIMPLE_COMPONENT_PORT" 
    ;;
    *)
    echo "Failure: unknown action $action"
    exit 1
    ;;
esac
echo "Action $action complete for $name"
```
</details>

### Building our invocation image
```sh
$ cd simple
$ docker build -t simple:1.0.0-invoc -f cnab/build/Dockerfile .
[+] Building 0.7s (7/7) FINISHED
 => [internal] load .dockerignore                                                                                                                                               0.0s
 => => transferring context: 2B                                                                                                                                                 0.0s
 => [internal] load build definition from Dockerfile                                                                                                                            0.0s
 => => transferring dockerfile: 109B                                                                                                                                            0.0s
 => [internal] load metadata for docker.io/library/alpine:3.9.3                                                                                                                 0.3s
 => CACHED [1/2] FROM docker.io/library/alpine:3.9.3@sha256:28ef97b8686a0b5399129e9b763d5b7e5ff03576aa5580d6f4182a49c5fe1913                                                    0.0s
 => [internal] load build context                                                                                                                                               0.1s
 => => transferring context: 86B                                                                                                                                                0.0s
 => [2/2] COPY cnab/app/run /cnab/app/run                                                                                                                                       0.1s
 => exporting to image                                                                                                                                                          0.0s
 => => exporting layers                                                                                                                                                         0.0s
 => => writing image sha256:03addb3d4bab4e50084921db9a14decf1bc7a86fe18d62c7e7676c33665e8e5c                                                                                    0.0s
 => => naming to docker.io/library/simple:1.0.0-invoc                                                                                                                           0.0s
```

We have now a `bundle.json` AND an invocation image `simple:1.0.0-invoc`! Congratulations you have made your first CNAB bundle!!! :tada:
It's time to play with it using Docker App!

## Use Docker App as a standard CNAB runtime

Docker App is a standard CNAB runtime. It can execute the following actions on any CNAB bundle:
- install
- upgrade
- io.cnab.status
- uninstall

**EXERCISE**: Let's play a little with our bundle, try to install, status, upgrade with another port, status again, then uninstall.

<details>
    <summary>Solution</summary>

* **install** the application:
```sh
$ docker app install bundle.json --name my-simple-app
Install action
Bundle simple version 1.0.0
Action install complete for my-simple-app
Application "my-simple-app" installed on context "dev"
```

* **status** command queries the `status` action and shows the parameters and the last action result: 
```sh
$ docker app status my-simple-app
INSTALLATION
------------
Name:        my-simple-app
Created:     28 seconds
Modified:    27 seconds
Revision:    01D9JYYKCN972PDF37DYTH55DB
Last Action: install
Result:      SUCCESS

APPLICATION
-----------
Name:      simple
Version:   1.0.0
Reference:

PARAMETERS
----------
simple_component_port: 8080

STATUS
------
Status action
current simple_component_port 8080
Action io.cnab.status complete for my-simple-app
```

* **upgrade** the application using a new port
```sh
$ docker app upgrade my-simple-app --set simple_component_port=9090
Upgrade action
Action upgrade complete for my-simple-app
Application "my-simple-app" upgraded on context "dev"
```

* **status** again, the last action is upgrade, and the listed parameter port should have changed too
```sh
$ docker app status my-simple-app
INSTALLATION
------------
Name:        my-simple-app
Created:     About a minute
Modified:    15 seconds
Revision:    01D9JZ0K8VRKJZ75HVFAMGA01D
Last Action: upgrade
Result:      SUCCESS

APPLICATION
-----------
Name:      simple
Version:   1.0.0
Reference:

PARAMETERS
----------
simple_component_port: 9090

STATUS
------
Status action
current simple_component_port 9090
Action io.cnab.status complete for my-simple-app
```

* **uninstall** the application
```sh
$ docker app uninstall my-simple-app
Uninstall action
Action uninstall complete for my-simple-app
Application "my-simple-app" uninstalled on context "dev"
```
</details>
