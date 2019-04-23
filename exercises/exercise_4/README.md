# Exercise - Install with docker-app

> **Time**: Approximately 10 minutes

With this exerise you will learn how to deploy an application package using `docker-app`.

## Bundle the application

Deployment is named `install` with docker-app, but before installation, docker-app needs a step to prepare the `bundle`. This step generates two artifacts:
- A manifest file `bundle.json`
- an invocation image

| Sources             | Compile             | Binary                         | Instantiate          | Runtime   |
|---------------------|---------------------|--------------------------------|----------------------|-----------|
| DockerFile          | `docker build`      | Image                          | `docker run`         | Container |
| Application Package | `docker-app bundle` | Bundle.json + Invocation image | `docker-app install` | Stack     |

* **Bundle** the `hello` application

```sh
$ docker-app bundle hello.dockerapp
Invocation image "hello:0.1.0-invoc" successfully built

/workshop $ ls
bundle.json  docker-compose.yml  hello.dockerapp

/workshop $ docker image ls | grep hello
hello                                 0.1.0-invoc         b4582d6d7187        3 seconds ago       40.1MB
```

The `bundle.json` has appeared in the dockerapp directory and the invocation image in the local image store.

### Bundle.json

The `bundle.json` is a representation of our application package in a `JSON` portable way.

```sh
/workshop $ cat bundle.json
```
```json
{
        "name": "hello",
        "version": "0.1.0",
        "description": "Hello DockerCon application",
        "invocationImages": [
                {
                        "imageType": "docker",
                        "image": "hello:0.1.0-invoc"
                }
        ],
        "images": null,
        "actions": {
                "inspect": {
                        "Modifies": false
                }
        },
        "parameters": {
                "docker.kubernetes-namespace": {
                        "type": "string",
                        "defaultValue": "",
                        "required": false,
                        "metadata": {
                                "description": "Namespace in which to deploy"
                        },
                        "destination": {
                                "path": "",
                                "env": "DOCKER_KUBERNETES_NAMESPACE"
                        }
                },
                "docker.orchestrator": {
                        "type": "string",
                        "defaultValue": "",
                        "allowedValues": [
                                "",
                                "swarm",
                                "kubernetes"
                        ],
                        "required": false,
                        "metadata": {
                                "description": "Orchestrator on which to deploy"
                        },
                        "destination": {
                                "path": "",
                                "env": "DOCKER_STACK_ORCHESTRATOR"
                        }
                },
                "hello.port": {
                        "type": "string",
                        "defaultValue": "8080",
                        "required": false,
                        "metadata": {},
                        "destination": {
                                "path": "",
                                "env": "docker_param1"
                        }
                },
                "hello.text": {
                        "type": "string",
                        "defaultValue": "Hello DockerCon",
                        "required": false,
                        "metadata": {},
                        "destination": {
                                "path": "",
                                "env": "docker_param2"
                        }
                }
        },
        "credentials": {
                "docker.context": {
                        "path": "/cnab/app/context.dockercontext",
                        "env": ""
                }
        }
}
```

You can find all the metadata and parameters of our application:
- name
- description
- version
- parameters `hello.text` and `hello.port`

Plus other things needed to specify deployment context:
- credentials
- targeted orchestrator
- enventually kubernetes namespace

All these parameters are set at the deployment step, some are set by the users (parameters), some are set by docker-app.

### Invocation image

The `invocation image` is a regular image, producing a regular docker container. It contains everything needed to deploy your application:
- a binary running the deployment toward Swarm or Kubernetes.
- all the application package files + attachments

The advantages to pack the installation as an invocation image are multiple:
- it is self-sufficient and can be executed on any docker engine
- it is independant of the docker-app version your are using, as the runtime to deploy is embedded in the image. You will be able to re-deploy an old invocation-image without worrying about getting the old docker-app binary.
- the invocation image can be shared on the hub or on any private registry (we will see that in the next exercise)

**NOTE:** The `invocation image` is meant to run on a local docker engine, but targeting a remote orchestrator. Of course it can be also executed on the orchestrator context.

## Actions

The invocation image answers to five actions:
- `install`
- `status`
- `upgrade`
- `uninstall`
- `inspect`

Let's go through the first four, we already know the fifth.

### Install

`Usage:  docker-app install [<bundle name>] [options] [flags]`

`install` command takes a required bundle name and let you choose your installation name with `--name` flag.

```sh
/workshop $ docker-app install hello.dockerapp --name my-hello-installation
Creating network my-hello-installation_default
Creating service my-hello-installation_hello

/workshop $ docker stack ls
NAME                    SERVICES            ORCHESTRATOR
my-hello-installation   1                   Swarm
```

**NOTE:** Don't mix up `bundle name` and `installation name`. By default, if you don't specify the installation name, it will use the bundle name, but be carefull as the installation name must be `hostname` compatible (`_` is forbidden).

You can of course use the same flags `--set` and `--parameters-files` as with the `render` command.

* **`install`** another `hello` application, modifying the ports and the installation name

```sh
$ docker-app install hello.dockerapp --name my-hello-prod-installation -f prod-parameters.yml
Creating network my-hello-prod-installation_default
Creating service my-hello-prod-installation_hello

$ docker stack ls
NAME                         SERVICES            ORCHESTRATOR
my-hello-installation        1                   Swarm
my-hello-prod-installation   1                   Swarm
```

**NOTE:** `install` can take any dockerapp single-file or directory. In that case it will automatically bundle the application then install it. `install` can also deploy directly a previously bundled `bundle.json`.

* **`install`** using the bundle.json previously built

```sh
$ docker-app install bundle.json --name my-app-with-bundle -s hello.port=8181
Creating network my-app-with-bundle_default
Creating service my-app-with-bundle_hello
```

### Status

`Usage:  docker-app status <installation-name> [flags]`

`status` command takes an installation name and displays informations about deployment or upgrade, service per service.

* **Display** the status of each installation

```sh
$ docker-app status my-hello-installation
ID                  NAME                          MODE                REPLICAS            IMAGE                        PORTS
qspyvbdxf1js        my-hello-installation_hello   replicated          1/1                 hashicorp/http-echo:latest   *:8080->8080/tcp

$ docker-app status my-hello-prod-installation
ID                  NAME                               MODE                REPLICAS            IMAGE                        PORTS
5d1glvey8b1l        my-hello-prod-installation_hello   replicated          1/1                 hashicorp/http-echo:latest   *:80->80/tcp
```

### Upgrade

`Usage:  docker-app upgrade <installation-name> [options] [flags]`

`upgrade` command lets you modify your installation, for example to change a port or to scale a service.

* **`upgrade`** the `my-hello-installation` installation and change the `hello.text`

```sh
$ docker-app upgrade my-hello-installation -s hello.text="Hello upgrade"
Updating service my-hello-installation_hello (id: qspyvbdxf1jsh5hj95er7x598)
```

* **Check** the text has been updated on port `8080`

### Uninstall

`Usage:  docker-app uninstall <installation-name> [flags]`

`uninstall` command removes your installation.

* **`uninstall`** all the previous installations.

```sh
$ docker-app uninstall my-hello-installation
Removing service my-hello-installation_hello
Removing network my-hello-installation_default
```

## Summary

* The bundle is a new way to install your application. It is self-contained, including the application files and the deploy runtime binaries.
* The invocation image can be stored and shared in a registry
* The bundle.json is a portable way to define the application
