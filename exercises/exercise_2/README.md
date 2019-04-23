# Exercise - Discover Docker Application Package

> **Time**: Approximately 20 minutes

During this exercise we will introduce `Docker Application Package` and its dedicated tool `docker-app`.
`Application Package` is a construction above `Compose file` to improve the application lifecycle and workflow, from development to test to production.

An application package is a set of 3 documents, plus custom files we name `attachments`:
* a `metadata.yml` file describing the application metadata (name, version, description, ...)
* a `docker-compose.yml` file describing the application structure
* a `parameters.yml` file with key/value parameters (we will see this part in the next exercise)

First things first, let's initialize our first application package, using the previous `words` exercise.

## Initialize the Application Package

```sh
$ docker-app init --help

Usage:  docker-app init <app-name> [-c <compose-file>] [-d <description>] [-m name:email ...] [flags]

Start building a Docker application. Will automatically detect a docker-compose.yml file in the current directory.

Options:
  -c, --compose-file string      Initial Compose file (optional)
  -d, --description string       Initial description (optional)
  -m, --maintainer stringArray   Maintainer (name:email) (optional)
  -s, --single-file              Create a single-file application
```

* Use **docker-app init** command to create the application package from the compose file
```sh
workshop $ cd words
words $ docker-app init myapp -d "My word application" -m dapworkshop:dapworkshop@docker.com
words $ tree .
.
|-- docker-compose.yml
`-- myapp.dockerapp
    |-- docker-compose.yml
    |-- metadata.yml
    `-- parameters.yml

1 directory, 4 files
```

This command creates a new directory with the name of your app + `.dockerapp` suffix. It copies the initial compose file and generates `metadata.yml` with the given information, plus an `parameters.yml` file.

An application package can also be a single file, using the multi-yaml document feature. It's very handy for sharing an app!

* **Merge** the application package to the one-file version

```sh
words $ docker-app merge myapp -o myapp-merged.dockerapp
words $ cat myapp-merged.dockerapp
```
```yaml
# Version of the application
version: 0.1.0
# Name of the application
name: myapp
# A short description of the application
description: My word application
# Namespace to use when pushing to a registry. This is typically your Hub username.
#namespace: myhubusername
# List of application maintainers with name and email for each
maintainers:
  - name: dapworkshop
    email: dapworkshop@docker.com

---
version: "3.7"
services:
    web:
     image: dockerdemos/lab-web
     ports:
      - "33000:80"

    words:
     image: dockerdemos/lab-words
     deploy:
       replicas: 5
       endpoint_mode: dnsrr

    db:
     image: dockerdemos/lab-db

---
{}
```

**NOTE:** If you don't specify the `-o`, the merge will transform your application to a one-file version in place, removing the 3 files. The same behavior applies to the `split` command.

* **Split** the one-file application package to a directory
```sh
words $ docker-app split myapp-merged.dockerapp -o myapp-split
words $ tree.
.
|-- docker-compose.yml
|-- myapp-merged.dockerapp
|-- myapp-split
|   |-- docker-compose.yml
|   |-- metadata.yml
|   `-- parameters.yml
`-- myapp.dockerapp
    |-- docker-compose.yml
    |-- metadata.yml
    `-- parameters.yml

2 directories, 8 files
```

* Let's **clean** our workspace and remove the split and merged versions, keep only `myapp.dockerapp` directory
```sh
words $ rm -rf myapp-split myapp-merged.dockerapp
```

## Inspect your application

`docker-app` comes with another very handy command: `inspect`. With it you can display all important informations without having to read the `YAML` yourself.

* **Inspect** your application package
```sh
words $ docker-app inspect myapp
myapp 0.1.0

Maintained by: dapworkshop <dapworkshop@docker.com>

My word application

Services (3) Replicas Ports Image
------------ -------- ----- -----
db           1              dockerdemos/lab-db
web          1        33000 dockerdemos/lab-web
words        3              dockerdemos/lab-words
```

`inspect` command will
* pretty print the metadata
* list the services
* but also volumes
* networks
* secrets 
* attachments
* parameters

**NOTE**: All `docker-app` commands have multiple ways to take an application as an argument:
```sh
# You can omit it if current directory is a `*.dockerapp`
myapp.dockerapp $ docker-app inspect
# or if there is only one `*.dockerapp` (file or directory) in the current directory
words $ docker-app inspect
# you can reference it by its name only `path/to/myapp`
workshop $ docker-app inspect words/myapp
# or by its full path `path/to/myapp.dockerapp`
workshop $ docker-app inspect words/myapp.dockerapp
```

* **Try** to inspect the application from different working directories.

## Validate your application

The `validate` command checks everything is ok with your application:
* Compose file is valid
* Required Metadata are filled (name, version) and have the good type
* Default parameters

* **Validate** your application
```sh
words $ docker-app validate myapp
words $ echo $?
0
```

Let's modify a bit this app:
* **Comment** the version field in `myapp.dockerapp/metadata.yml` using `#`
* **Add** some garbage characters to the name: `name: myapp$%%^`
* **Validate** the application
```sh
words $ docker-app validate myapp
Error: failed to validate metadata:
- name: Does not match format 'hostname'
- version: version is required
words $ echo $?
1
```
* **Fix** `myapp.dockerapp/metadata.yml`

* **Comment** the `services:` line in `myapp.dockerapp/docker-compose.yml`
* **Validate** the application
```sh
words $ docker-app validate myapp
Error: failed to load composefiles: failed to parse Compose file version: "3.7"
#services:
    web:
     image: dockerdemos/lab-web
     ports:
      - "33000:80"

    words:
     image: dockerdemos/lab-words
     deploy:
       replicas: 3
       endpoint_mode: dnsrr

    db:
     image: dockerdemos/lab-db

: yaml: line 2: did not find expected key

words $ echo $?
1
```
* **Fix** `myapp.dockerapp/docker-compose.yml`

**Summary**
* `docker-app` let's you define metadata and parameters on top of a compose file
* `inspect` displays all important informations in a compact way
* `validate` checks everything is ok in the application. You can use it as a CI step.
* `split`/`merge` transforms your application to a multiple-files/one-file. The one-file version is easy to share.
