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
2. Create the following hierarchy (just touching empty files for now):

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

### Create the `bundle.json`

We will create a simple `bundle.json`, declaring one invocation image and one parameter.

1. Edit `simple/bundle.json`
1. Add an invocation image with the following name `username/simple:1.0.0-invoc` (:warning: use your own docker hub account).
1. Add an `integer` parameter called `port` and available as an environment variable (`$PORT_PARAMETER`). Add its parameter definition.

<details>
    <summary>Solution</summary>

```json
{
    "name": "simple",
    "version": "1.0.0",
    "description": "a very simple CNAB Bundle",
    "maintainers": [
        {
            "name": "DAP Workshop",
            "email": "dapworkshop@docker.com"
        }
    ],
    "invocationImages": [
        {
            "imageType": "docker",
            "image": "username/simple:1.0.0-invoc"
        }
    ],
    "definitions":{ 
      "port":{ 
         "maximum":65535,
         "minimum":1024,
         "type":"integer"
      }
    },
    "parameters":{ 
      "port":{ 
         "definition":"port",
         "destination":{ 
            "env":"PORT_PARAMETER"
         }
      }
   },
   "schemaVersion":"v1.0.0"
}
```
</details>
<br/>

### Create the CNAB `run` executable

We will create a simple bash script which can react to CNAB actions.

1. Edit `simple/cnab/app/run`
1. Read the environment variable `CNAB_ACTION` and print a specific action result on the standard output. It must also fail if the action is unknown. According to our `bundle.json`, it must understand `install`/`upgrade`/`uninstall` actions.
1. For each action, print the installation name.
1. The `install` action must print the bundle name and the bundle version.
1. The `install` and `upgrade` actions must print the `port` parameter (in `$PORT_PARAMETER`).

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
    echo "Port $PORT_PARAMETER"
    ;;
    uninstall)
    echo "Uninstall action"
    ;;
    upgrade)
    echo "Upgrade action"
    echo "Upgraded port $PORT_PARAMETER"
    ;;
    *)
    echo "Failure: unknown action $action"
    exit 1
    ;;
esac
echo "Action $action complete for $name"
```
</details>

### Create the `Dockerfile` for the invocation image

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

### Build our invocation image
```sh
$ cd simple
$ docker build -t username/simple:1.0.0-invoc -f cnab/build/Dockerfile .
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

## Push the bundle to Docker Hub

Now we need push the invocation image on the hub to make it available, before pushing the bundle itself.
```console
$ docker push username/simple:1.0.0-invoc
```
Then we will use `cnab-to-oci`, the standard tool to push and pull CNAB bundles to a registry.
**NOTE**: Sources can be found here https://github.com/docker/cnab-to-oci
You will find the tool in your home directory: `~/cnab-to-oci`

```console
$ ./cnab-to-oci push simple/bundle.json --auto-update-bundle --target username/simple:v0.1
Starting to copy image username/simple:1.0.0-invoc...
Completed image username/simple:1.0.0-invoc copy
Pushed successfully, with digest "sha256:f0bbfb67115b8517f3880a00f7b179208961482a2c7cb4591c6407edc72ee971"
```
:tada: Our bundle has been pushed on the Docker Hub and is now available for docker app!
## Use Docker App as a standard CNAB runtime

Docker App is a standard CNAB runtime. It can execute the following actions on any CNAB bundle:
- install
- upgrade
- uninstall

**EXERCISE**: Let's play a little with our bundle, try to run, upgrade with another port, then remove the application.

<details>
    <summary>Solution</summary>

* **Run** the application:
```sh
$ docker app run username/simple:v0.1 --name simple_app  --set port=4242
Unable to find application image "username/simple:v0.1" locally
Pulling from registry...
Install action
Bundle simple version 1.0.0
Port 4242
Action install complete for simple_app
Application "simple_app" installed on context "default"
```

* **upgrade** the application using a new port
```sh
$ docker app upgrade simple_app --set port=9090
Upgrade action
Upgraded port 8080
Action upgrade complete for simple_app
Application "simple_app" upgraded on context "dev"
```

* **rm** the application
```sh
$ docker app rm simple_app
Uninstall action
Action uninstall complete for simple_app
Application "simple_app" uninstalled on context "dev"
```
</details>
