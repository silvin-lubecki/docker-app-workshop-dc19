# Exercise - Configure your application with parameters

> **Time**: Approximately 20 minutes

## Introducing parameters

During this exercise we will learn how add parameters and use them for deployment.

`docker-app` makes the already existing `Compose` variable substitutions system easy to use.
Just replace any part of the compose file with a variable using this form: `${path.to.my-variable}`
`docker-app` will seek into the `parameters.yml` file, which is a simple key/value YAML file, to find the default value.

Here is the corresponding `parameters.yml` file:
```yaml
path:
    to:
        my-variable: myvalue
        other-variable: othervalue
```

**NOTE:** All the application parameters will be displayed using the `inspect` command.

## Add parameters to an existing `docker-compose.yml` file

We will re-use the first compose file we wrote, with the `hello` service.

* **Go back** to `/workshop` and create a single-file `hello` application using the previous `docker-compose.yml`
```sh
words $ cd /workshop
/workshop $ docker-app init hello --compose-file docker-compose.yml --description "Hello DockerCon application" --single-file
```
It produces a `hello.dockerapp` file which should be like the following:
```yaml
# This section contains your application metadata.
# Version of the application
version: 0.1.0
# Name of the application
name: hello
# A short description of the application
description: Hello DockerCon application
# Namespace to use when pushing to a registry. This is typically your Hub username.
#namespace: myhubusername
# List of application maintainers with name and email for each
#maintainers:
#  - name: John Doe
#    email: john@doe.com

---
# This section contains the Compose file that describes your application services.
version: '3.7'
services:
  hello:
    image: hashicorp/http-echo:latest
    command: ["-text", "Hello DockerCon", "-listen",":8080"]
    ports:
     - 8080:8080

---
# This section contains the default values for your application parameters.
{}
```

* **Edit** the `hello.dockerapp` file and replace the "Hello DockerCon" text with a variable `${hello.text}` and add the variable as a parameter in the `parameters` section
* **Use `validate`** while you edit your application to check everything is ok
* **`inspect`** your application, now the parameters section is displayed

```sh
$ docker-app inspect hello
hello 0.1.0

Hello DockerCon application

Service (1) Replicas Ports Image
----------- -------- ----- -----
hello       1        8080  hashicorp/http-echo:latest

Parameter (1) Value
------------- -----
hello.text    Hello DockerCon
```

* Replace all the `8080` ports with a `${hello.port}` variable, and add it too to the `parameters` section
* `validate` then `inspect` the application, it displays the `hello.port` parameter

**NOTE:** metadata are available as read-only variables under the `app` prefix. You can use them in your compose file:
- `${app.name}`
- `${app.version}`
- `${app.description}`

**NOTE:** the `parameters` section **MUST** define all the variables with a default value. If any parameter is missing, an error will occur on `validate` or `inspect` commands.

* **Comment** one of the two parameters, then `inspect` or `validate` the application. Don't forget to uncomment the parameter.
```sh
$ docker-app validate hello
Error: failed to load Compose file: invalid interpolation format for services.hello.command.[]: "required variable hello.text is missing a value". You may need to escape any $ with another $.
```

## Render an application to a compose file

The `render` command will produce a compose file, substituting all the variables with the default values.

* **`render`** the hello application
```sh
$ docker-app render hello
```
```yaml
version: "3.7"
services:
  hello:
    command:
    - -text
    - Hello DockerCon
    - -listen
    - :8080
    image: hashicorp/http-echo:latest
    ports:
    - mode: ingress
      target: 8080
      published: 8080
      protocol: tcp
```
* **Modify** the `hello.text` parameter in the `parameters` section and re-render
* **Replace** the `${hello.text}` variable by `${app.description}` and re-render, then revert it
* **Save** your rendered compose file using the `--output` flag
```sh
$ docker-app render hello --output hello.yml
```

## Override the parameters

You can also override all the variables using the command line, or create another parameters file, with other values, targeting another environment.

* **Override** the port using the `--set` flag with the `render` command
```sh
$ docker-app render hello --set hello.port=8181
```
```yaml
version: "3.7"
services:
  hello:
    command:
    - -text
    - Hello DockerCon
    - -listen
    - :8181
    image: hashicorp/http-echo:latest
    ports:
    - mode: ingress
      target: 8181
      published: 8181
      protocol: tcp
```

* **Edit** a new `prod-parameters.yml` file and add different values in it for `hello.text` and `hello.port` variables
* **`render`** the application using this new settings file and the flag `--parameters-files`
```sh
$ cat prod-parameters.yml
hello:
    text: Hello Workshop
    port: 80 
$ docker-app render hello --parameters-files prod-parameters.yml 
version: "3.7"
services:
  hello:
    command:
    - -text
    - Hello Workshop
    - -listen
    - :80
    image: hashicorp/http-echo:latest
    ports:
    - mode: ingress
      target: 80
      published: 80
      protocol: tcp
```
**NOTE:** Parameters can be overridden using a mix of files or command line parameters. The parameters files doesn't need to define all the variables, a subset is enough. The precedence is the following:
- command line parameters, last one has precedence on the others
- files
- default values in the parameters section.

* **Play** with `render` command and mix `--set` and `--parameters-files`
```sh
$ docker-app render hello -s hello.text="Hello Moby" -f prod-parameters.yml
```

**NOTE:** these flags work the same way with `inspect`, `validate` or `install` commands.

* **Try** the same mix of parameters with `inspect`
```sh
$ docker-app inspect hello -f prod-parameters.yml
hello 0.1.0

Hello DockerCon application

Service (1) Replicas Ports Image
----------- -------- ----- -----
hello       1        80    hashicorp/http-echo:latest

Parameters (2) Value
-------------- -----
hello.port     80
hello.text     Hello Workshop
```

## Deploy the rendered compose file

The rendered compose file can be directly injected to `docker-compose up` or `docker stack deploy` commands to deploy your application.

* **Deploy** using `docker stack deploy`, the `-` references the standard input
```sh
$ docker-app render hello | docker stack deploy my-hello-app -c -
Creating network my-hello-app_default
Creating service my-hello-app_hello
```
* **Open** a browser at the `8080` port, it displays `Hello DockerCon`
* **Re-deploy** changing the text
```sh
$ docker-app render hello -s hello.text="Hello Moby" | docker stack deploy my-hello-app -c -
```
* **Refresh** your browser and check the message
* **Deploy** another application using the production parameters, it should open it on port `80`
```sh  
$ docker-app render hello -f prod-parameters.yml | docker stack deploy my-prod-app -c -
```
* **Remove** the two stacks using `docker stack rm`

**NOTE:** With the parameters, you can now easily scale your application with `docker stack`.

**Bonus Exercise:** Add parameterized `deploy.replicas` to the `hello` service and use it to scale up and down with the `render` command and `--set hello.replicas=X`

## Summary

- With parameters you can use the same compose file and target multiple environments (dev/test/staging/prod/...) 
- Render let's you check what will be the exact compose-file deployed
- You can pipe the result of a render to your current workflow (`docker-compose`, `docker stack deploy`) and still use some of the benefits of docker-app

Here is a generic application package template:

```yaml
version: 0.1.0
name: base
description: A generic application template which takes an image as a parameter
maintainers:
  - name: garethr
    email: garethr@docker.com

---
version: '3.7'
services:
  app:
    image: ${image}
    ports:
     - ${port}:${port}
    deploy:
      replicas: ${replicas}
      resources:
        limits:
          memory: ${memory}

---
image: 
port: 8080
replicas: 1
memory: 128MB
```
