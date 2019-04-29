# Exercise 6 - CNAB Under the Hood

> **Time**: Approximately 15 minutes
> **Difficulty**: Medium

## Table of Contents
1. [What is CNAB](#what-is-cnab)
1. [Create a CNAB Bundle from scratch](#create-a-cnab-bundle-from-scratch)
1. [Docker App CNAB Runtime](#docker-app-cnab-runtime)

## Exercise Objectives

By the end of this exercise you will:

- Have a first glance on the new CNAB Specifications
- Learn how to create a CNAB from scratch
- Learn how to use Docker App as a CNAB client

## What is CNAB

CNAB, or [Cloud Native Application Bundle](https://cnab.io), is a new packaging standard describing how to manage an application lifecyle, from installation, upgrade, to uninstallation. It aims to become the `.deb` (or `.msi`) for your cloud applications.

**NOTE**: Concerning the status of this format, the specification is still at `Draft` level, but the core concepts are finalized and stabilized. There won't be any breaking change until the 1.0.0 release of the specifications, which will come very soon. By the way, **contributions are welcome**!

Before we start digging into the format, here are some CNAB key terms (shamelessly copied from the [specifications](https://github.com/deislabs/cnab-spec/blob/master/100-CNAB.md)):
> * **Application**: The functional unit composed by the components described in a bundle. This MAY be comprised of a mixture of containers, VMs, IaaS and PaaS definitions, and other services, as * well as instructions for orchestrators and service frameworks.
> * **Bundle**: the collection of CNAB data and metadata necessary for installing an application on the designated cloud services.
> * **Bundle definition**: The information about a bundle, its parameters, credentials, images, and usage
> * **bundle.json**: The unsigned JSON-encoded representation of a bundle definition.
> * **Image**: Used generically, a container image (e.g. OCI images) or a VM image.
> * **Invocation Image**: The image that contains the bootstrapping and installation logic for the bundle
> * **Registry**: A storage and retrieval service for CNAB objects.
> * **Claim**: A stored installation result
>
> Also, when referencing tooling, we use the following terms:
> * **CNAB runtime** or runtime: A program capable of reading a CNAB bundle and executing it
> * **CNAB builder** or builder: A program that can assemble a CNAB bundle
> * **bundle tooling**: Programs or tooling that generate CNAB bundle content

To summarize, a CNAB Bundle is the sum of two artifacts:
- a `bundle.json` file defining the application
- multiple invocation images, which can consist of either Docker images or Virtual Machine images

### Bundle.json

The first artifact is the bundle file, which defines the following parts:
- Metadata: application name, version, description, maintainers
- List of invocation images
- List of component images (compose services in our case)
- Custom actions: every bundle must implement the 3 mandatory actions **install**/**upgrade**/**uninstall**
- Parameters for these actions
- Credentials

Here is an example of a simple `bundle.json`:

```json
{
	"name": "simple",
	"version": "1.0.0",
	"description": "a very simple application",
	"maintainers": [
		{
			"name": "DAP Workshop",
			"email": "dapworkshop@docker.com"
		}
	],
	"invocationImages": [
		{
			"imageType": "docker",
			"image": "simple:1.0.0-invoc"
		}
	],
	"images": {
		"api": {
			"imageType": "docker",
			"image": "simple-component:1.0.0",
			"description": "simple component"
		}
	},
	"actions": {
		"io.cnab.status": {}
	},
	"parameters": {
		"simple_component_port": {
			"type": "string",
			"defaultValue": "8080",
			"destination": {
				"env": "SIMPLE_COMPONENT_PORT"
			}
		}
	},
	"credentials": {}
}
```

### Invocation image

**NOTE** During this exercise, we will only focus on Docker base invocation images.

The second artifact is the invocation image, which is a simple Docker image that is able to install the application. It's not the application itself; see it as the installer binary.

It must:
- contain a simple executable file at the path **`/cnab/app/run`**
- understand and react to actions declared in the `bundle.json`

Some parts are injected dynamically in the container by the runtime, like the credentials or the parameters.

When the `CNAB runtime` executes an action, it will follow these steps:
1. Parse and validate the bundle.json file
1. Resolve and validate (type, allowed values, required) the parameters
1. Create a container using the specified invocation image
1. Create environment variables for the installation name (`CNAB_INSTALLATION_NAME`), bundle name (`CNAB_BUNDLE_NAME`), bundle version (`CNAB_BUNDLE_VERSION`), and the action to execute (`CNAB_ACTION`)
1. Either create environment variables or mount files for parameters
1. Mount the credentials (using `tmpfs`) to the specified paths
1. Run the container and execute `/cnab/app/run`
1. Display the container's stdout/stderr
1. Create or Update a claim to store the action result (`SUCCESS` or `FAILURE`), if it is not stateless

## Create a CNAB Bundle from scratch

Let's put our hands on what we just saw and create our first CNAB!

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
``

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

## Docker App CNAB Runtime

### Use Docker App as a standard CNAB runtime

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
$ docker app install cnab/bundle.json --name my-simple-app
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

### Docker App CNAB Architecture

Now lets dig a little into docker app CNAB implementation. We will use for that our previous `voting-app` application package. Try the following command:

```sh
$ cd voting-app.dockerapp
$ docker app bundle
Invocation image "voting-app:0.1.0-invoc" successfully built

$ docker image ls | grep voting-app
voting-app  0.1.0-invoc  84b5475a8069  7 minutes ago  49.1MB
```
It just generated a `bundle.json` and an invocation image.
Let's take a look at the invocation image:

```sh
$ docker run -it --rm voting-app:0.1.0-invoc /bin/sh
/cnab/app $ ls -la
total 42028
drwxr-xr-x    1 root     root          4096 Apr 28 22:17 .
drwxr-xr-x    1 root     root          4096 Apr 28 21:46 ..
-rw-r--r--    1 root     root            10 Jan  1  1970 .dockerignore
-rwxr-xr-x    1 root     root      43020288 Apr 28 21:46 run
drwxr-xr-x    2 root     root          4096 Apr 28 22:17 voting-app.dockerapp
```

Docker app generated an invocation `FROM` a base docker app invocation image, containing the `run` binary. It copied your application package and all the attachments. The `run` backend knows how to install your package using `docker stack deploy`, and even `render` or `inspect` it. This `bundle` command is executed on-the-fly by each command, without you noticing it.

**NOTE**: As the backend and the package are stored together in the invocation image, it will always be able to install the application, even if there is a breaking change in the Docker App package format.

Let's check the generated `bundle.json`:

<details>
    <summary>Content</summary>

**TODO** real json from voting-app
```json
{
	"name": "voting-app",
	"version": "0.1.0",
	"description": "Voting App",
	"maintainers": [
		{
			"name": "dapworkshop",
			"email": "dapworkshop@docker.com"
		}
	],
	"invocationImages": [
		{
			"imageType": "docker",
			"image": "voting-app:0.1.0-invoc"
		}
	],
	"images": {
		"db": {
			"imageType": "docker",
			"image": "postgres:9.4",
			"description": "postgres:9.4"
		},
		"redis": {
			"imageType": "docker",
			"image": "redis:alpine",
			"description": "redis:alpine"
		},
		"results": {
			"imageType": "docker",
			"image": "mikesir87/examplevotingapp_result",
			"description": "mikesir87/examplevotingapp_result"
		},
		"vote": {
			"imageType": "docker",
			"image": "mikesir87/examplevotingapp_vote",
			"description": "mikesir87/examplevotingapp_vote"
		},
		"worker": {
			"imageType": "docker",
			"image": "dockersamples/examplevotingapp_worker",
			"description": "dockersamples/examplevotingapp_worker"
		}
	},
	"actions": {
		"com.docker.app.inspect": {
			"stateless": true
		},
		"com.docker.app.render": {
			"stateless": true
		},
		"com.docker.app.status": {}
	},
	"parameters": {
		"com.docker.app.kubernetes-namespace": {
			"type": "string",
			"defaultValue": "",
			"metadata": {
				"description": "Namespace in which to deploy"
			},
			"destination": {
				"env": "DOCKER_KUBERNETES_NAMESPACE"
			},
			"apply-to": [
				"install",
				"upgrade",
				"uninstall",
				"com.docker.app.status"
			]
		},
		"com.docker.app.orchestrator": {
			"type": "string",
			"defaultValue": "",
			"allowedValues": [
				"",
				"swarm",
				"kubernetes"
			],
			"metadata": {
				"description": "Orchestrator on which to deploy"
			},
			"destination": {
				"env": "DOCKER_STACK_ORCHESTRATOR"
			},
			"apply-to": [
				"install",
				"upgrade",
				"uninstall",
				"com.docker.app.status"
			]
		},
		"com.docker.app.render-format": {
			"type": "string",
			"defaultValue": "yaml",
			"allowedValues": [
				"yaml",
				"json"
			],
			"metadata": {
				"description": "Output format for the render command"
			},
			"destination": {
				"env": "DOCKER_RENDER_FORMAT"
			},
			"apply-to": [
				"com.docker.app.render"
			]
		},
		"com.docker.app.share-registry-creds": {
			"type": "bool",
			"defaultValue": false,
			"metadata": {
				"description": "Share registry credentials with the invocation image"
			},
			"destination": {
				"env": "DOCKER_SHARE_REGISTRY_CREDS"
			}
		}
	},
	"credentials": {
		"com.docker.app.registry-creds": {
			"path": "/cnab/app/registry-creds.json"
		},
		"docker.context": {
			"path": "/cnab/app/context.dockercontext"
		}
	}
}
```
</details>
<br/>

Docker App translated the Application Package to the CNAB format:
- Metadata
- List of services images
- 3 custom actions: `render` / `inspect` / `status`
- All the parameters + custom parameters used by Docker App backend
- Some credentials like exported docker contexts

After each installation, the state is stored locally (result, filled parameters...) so you won't have to specify them each time you interact with the installation. Have a look to your local store:

**TODO** Get one at this step of the workshop
```sh
$ tree ~/.docker/app
/Users/silvin/.docker/app
├── bundles
│   └── docker.io
│       ├── library
│       │   └── myapp
│       │       └── _tags
│       │           └── mytag.json
│       └── slubecki
│           └── example
│               └── _tags
│                   ├── v1.0.0.json
│                   └── v2.0.0.json
├── credentials
└── installations
    ├── 37a8eec1ce19687d132fe29051dca629d164e2c4958ba141d5f4133a33f0688f
    │   ├── cnab-with-status.json
    │   ├── cnab-without-status.json
    │   └── toto.json
    └── 7ab9b39a15b5b065f62b7d99d0e69e8f90ad69308a6ae873a8bbe70255646867
        └── toto.json

14 directories, 7 files
``` 

You can see three stores:
- `bundle store`: filled with pulled bundles
- `credential store`: filled with your installation credentials
- `installation store`: filled with your current installations
