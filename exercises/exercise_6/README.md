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


## Docker App CNAB Architecture

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
		},
		"options.A": {
				"type": "string",
				"defaultValue": "Cats",
				"destination": {
						"env": "docker_param1"
				}
		},
		"options.B": {
				"type": "string",
				"defaultValue": "Dogs",
				"destination": {
						"env": "docker_param2"
				}
		},
		"redis.cpu_limit": {
				"type": "string",
				"defaultValue": "1",
				"destination": {
						"env": "docker_param3"
				}
		},
		"redis.memory_limit": {
				"type": "string",
				"defaultValue": "512M",
				"destination": {
						"env": "docker_param4"
				}
		},
		"redis.replicas": {
				"type": "string",
				"defaultValue": "1",
				"destination": {
						"env": "docker_param5"
				}
		},
		"results.cpu_limit": {
				"type": "string",
				"defaultValue": "1",
				"destination": {
						"env": "docker_param6"
				}
		},
		"results.memory_limit": {
				"type": "string",
				"defaultValue": "512M",
				"destination": {
						"env": "docker_param7"
				}
		},
		"results.replicas": {
				"type": "string",
				"defaultValue": "1",
				"destination": {
						"env": "docker_param8"
				}
		},
		"vote.cpu_limit": {
				"type": "string",
				"defaultValue": "1",
				"destination": {
						"env": "docker_param9"
				}
		},
		"vote.memory_limit": {
				"type": "string",
				"defaultValue": "512M",
				"destination": {
						"env": "docker_param10"
				}
		},
		"vote.replicas": {
				"type": "string",
				"defaultValue": "2",
				"destination": {
						"env": "docker_param11"
				}
		},
		"worker.cpu_limit": {
				"type": "string",
				"defaultValue": "1",
				"destination": {
						"env": "docker_param12"
				}
		},
		"worker.memory_limit": {
				"type": "string",
				"defaultValue": "512M",
				"destination": {
						"env": "docker_param13"
				}
		},
		"worker.replicas": {
				"type": "string",
				"defaultValue": "1",
				"destination": {
						"env": "docker_param14"
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

```sh
$ tree ~/.docker/app
/root/.docker/app
├── bundles
│   └── docker.io
│       └── dapworkshop
│           └── voting-app.dockerapp
│               └── _tags
│                   └── v0.0.1.json
├── credentials
    └── 7ab9b39a15b5b065f62b7d99d0e69e8f90ad69308a6ae873a8bbe70255646867
└── installations
    └── 7ab9b39a15b5b065f62b7d99d0e69e8f90ad69308a6ae873a8bbe70255646867
        └── voting-app.json

9 directories, 2 files
``` 

You can see three stores:
- `bundle store`: filled with pulled bundles
- `credential store`: filled with your installation credentials
- `installation store`: filled with your current installations
