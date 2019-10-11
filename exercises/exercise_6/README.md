# Exercise 6 - CNAB Under the Hood

> **Time**: Approximately 15 minutes
>
> **Difficulty**: Medium

## Table of Contents
1. [What is CNAB](#what-is-cnab)
1. [Docker App CNAB Architecture](#docker-app-cnab-architecture)

## Exercise Objectives

By the end of this exercise you will:

- Have a first glance on the new CNAB Specifications
- Learn how Docker App uses CNAB

## What is CNAB

CNAB, or [Cloud Native Application Bundle](https://cnab.io), is a new packaging standard describing how to manage an application lifecyle, from installation, upgrade, to uninstallation. It aims to become the `.deb` (or `.msi`) for your cloud applications.

Before we start digging into the format, here are some CNAB key terms (shamelessly copied from the [specifications](https://github.com/deislabs/cnab-spec/blob/master/100-CNAB.md)):
- **Application**: The functional unit composed by the components described in a bundle. This MAY be comprised of a mixture of containers, VMs, IaaS and PaaS definitions, and other services, as well as instructions for orchestrators and service frameworks.
- **Bundle**: the collection of CNAB data and metadata necessary for installing an application on the designated cloud services.
- **Bundle definition**: The information about a bundle, its parameters, credentials, images, and usage
- `bundle.json`: The unsigned JSON-encoded representation of a bundle definition.
- `bundle.cnab`: The signed JSON-encoded representation of a bundle definition.
- **Image**: Used generically, a container image (e.g. OCI images) or a VM image.
- **Invocation Image**: The image that contains the bootstrapping and installation logic for the bundle
- **Registry**: A storage and retrieval service for CNAB objects.

Also, when referencing tooling, the following terms are used:

- `CNAB runtime` or `runtime`: A program capable of reading a CNAB bundle and executing it
- `CNAB builder` or `builder`: A program that can assemble a CNAB bundle
- `bundle tooling`: Programs or tooling that generate CNAB bundle contents

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


## Docker App CNAB Architecture

Now lets dig a little into docker app CNAB implementation. We will use for that our previous `voting-app` application package. Build it:

```sh
$ docker app build voting-app.dockerapp
[+] Building 0.2s (6/6) FINISHED                                                                                 
 => [internal] load remote build context                                                                    0.0s
 => copy /context /                                                                                         0.1s
 => [internal] load metadata for docker.io/docker/cnab-app-base:v0.8.0-215-g3b9a6e3587                      0.0s
 => [1/2] FROM docker.io/docker/cnab-app-base:v0.8.0-215-g3b9a6e3587                                        0.0s
 => => resolve docker.io/docker/cnab-app-base:v0.8.0-215-g3b9a6e3587                                        0.0s
 => [2/2] COPY . .                                                                                          0.0s
 => exporting to image                                                                                      0.0s
 => => exporting layers                                                                                     0.0s
 => => writing image sha256:99ee2972e68ab62fa9bbb2d7ce410367285902af640a60b12b1d6455897c9aaf                0.0s
Successfully built service images
Successfully build 5a81892da45c0ea5a2722b53f9178bcab062a86772832183ed358d8e8986d984

$ docker image ls | grep voting-app
voting-app  0.1.0-invoc  84b5475a8069  7 minutes ago  49.1MB
```
It builds the invocation image, let's take a look at it:

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

After each build/installation, the state is stored locally (result, filled parameters...) so you won't have to specify them each time you interact with the installation. Have a look to your local store:

```sh
$ tree ~/.docker/app
/home/user/.docker/app
├── bundles
│   └── _ids
│       └── 083dc787b8438f72982e32a670dfcb31a4c18534986d7d35fb2d8b27d2546c90
│           └── bundle.json
├── credentials
│   └── 37a8eec1ce19687d132fe29051dca629d164e2c4958ba141d5f4133a33f0688f
└── installations
    └── 37a8eec1ce19687d132fe29051dca629d164e2c4958ba141d5f4133a33f0688f
        └── cool_chaum.json

7 directories, 2 files
``` 

You can see three stores:
- `bundle store`: filled with pulled bundles
- `credential store`: filled with your installation credentials
- `installation store`: filled with your current installations

Let's check the generated `bundle.json`:

<details>
    <summary>Content</summary>

```json
{
  "actions": {
    "com.docker.app.inspect": {
      "stateless": true
    },
    "com.docker.app.render": {
      "stateless": true
    },
    "io.cnab.status": {}
  },
  "credentials": {
    "com.docker.app.registry-creds": {
      "path": "/cnab/app/registry-creds.json"
    },
    "docker.context": {
      "path": "/cnab/app/context.dockercontext"
    }
  },
  "definitions": {
    "com.docker.app.inspect-format": {
      "default": "json",
      "description": "Output format for the inspect command",
      "enum": [
        "json",
        "pretty"
      ],
      "title": "Inspect format",
      "type": "string"
    },
    "com.docker.app.kubernetes-namespace": {
      "default": "",
      "description": "Namespace in which to deploy",
      "title": "Namespace",
      "type": "string"
    },
    "com.docker.app.orchestrator": {
      "default": "",
      "description": "Orchestrator on which to deploy",
      "enum": [
        "",
        "swarm",
        "kubernetes"
      ],
      "title": "Orchestrator",
      "type": "string"
    },
    "com.docker.app.render-format": {
      "default": "yaml",
      "description": "Output format for the render command",
      "enum": [
        "yaml",
        "json"
      ],
      "title": "Render format",
      "type": "string"
    },
    "com.docker.app.share-registry-creds": {
      "default": false,
      "description": "Share registry credentials with the invocation image",
      "title": "Share registry credentials",
      "type": "boolean"
    }
  },
  "description": "",
  "images": {
    "db": {
      "description": "postgres:9.4",
      "image": "postgres:9.4",
      "imageType": "docker"
    },
    "redis": {
      "description": "redis:alpine",
      "image": "redis:alpine",
      "imageType": "docker"
    },
    "results": {
      "description": "mikesir87/examplevotingapp_result",
      "image": "mikesir87/examplevotingapp_result",
      "imageType": "docker"
    },
    "vote": {
      "description": "mikesir87/examplevotingapp_vote",
      "image": "mikesir87/examplevotingapp_vote",
      "imageType": "docker"
    },
    "worker": {
      "description": "dockersamples/examplevotingapp_worker",
      "image": "dockersamples/examplevotingapp_worker",
      "imageType": "docker"
    }
  },
  "invocationImages": [
    {
      "contentDigest": "sha256:efb421588a6b18e363658e44b8efbb15342002abb27a1c9102a177bd09986497",
      "image": "voting-app:0.1.0-invoc",
      "imageType": "docker",
      "size": 47774887
    }
  ],
  "maintainers": [
    {
      "name": "djordjelukic"
    }
  ],
  "name": "voting-app",
  "parameters": {
    "com.docker.app.inspect-format": {
      "applyTo": [
        "com.docker.app.inspect"
      ],
      "definition": "com.docker.app.inspect-format",
      "destination": {
        "env": "DOCKER_INSPECT_FORMAT"
      }
    },
    "com.docker.app.kubernetes-namespace": {
      "applyTo": [
        "install",
        "upgrade",
        "uninstall",
        "io.cnab.status"
      ],
      "definition": "com.docker.app.kubernetes-namespace",
      "destination": {
        "env": "DOCKER_KUBERNETES_NAMESPACE"
      }
    },
    "com.docker.app.orchestrator": {
      "applyTo": [
        "install",
        "upgrade",
        "uninstall",
        "io.cnab.status"
      ],
      "definition": "com.docker.app.orchestrator",
      "destination": {
        "env": "DOCKER_STACK_ORCHESTRATOR"
      }
    },
    "com.docker.app.render-format": {
      "applyTo": [
        "com.docker.app.render"
      ],
      "definition": "com.docker.app.render-format",
      "destination": {
        "env": "DOCKER_RENDER_FORMAT"
      }
    },
    "com.docker.app.share-registry-creds": {
      "definition": "com.docker.app.share-registry-creds",
      "destination": {
        "env": "DOCKER_SHARE_REGISTRY_CREDS"
      }
    }
  },
  "schemaVersion": "v1.0.0-WD",
  "version": "0.1.0"
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
