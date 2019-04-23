# Exercise - Share your application

> **Time**: Approximately 10 minutes

With this exercise you will learn how to share your application.

Using the commands `push` and `pull`, you can share your bundle on any public or private registry.

`docker-app` will create a manifest and push the `bundle.json` with it. It will also push the invocation image and reference it from the manifest. This is a totally valid way to create an image, compatible with the OCI reference, and then with all the registries (`docker hub`, `docker registry` or the enterprise `docker trust registry`).

## `push` and `pull`

```sh
$ docker-app push --help

Usage:  docker-app push [<app-name>] [flags]

Push the application to a registry

Options:
      --insecure           Use insecure registry, without SSL
      --namespace string   Namespace to use (default: namespace in metadata)
      --repo string        Name of the remote repository (default: <app-name>.dockerapp)
  -t, --tag string         Tag to use (default: version in metadata)
```
**NOTE** about the namespace: it is defined as a registry hostname+ the organisation or the user.
* `--namespace=localhost:5000/myuser` to target a registry run locally
* `--namespace=my.private.registry/myteam` to target a remote registry
* `--namespace=myteam` to target the DockerHub, with organization or username `myteam`

Let's push the hello application to your own namespace (that's why you logged in docker hub during the first exercise).

```sh
$ docker-app push --namespace [myhublogin]
The push refers to repository [docker.io/dapworkshop/words]
a8e86457508f: Pushed
abac5e0b2197: Pushed
12ad74ab2cc9: Pushed
df64d3292fd6: Mounted from docker/cnab-app-base
0.1.0-invoc: digest: sha256:b92db3946b9c3750b31e744973b063cab1c9a8cf6e0e969ec1ba741ac414c477 size: 1157
Successfully pushed dapworkshop/words:0.1.0@sha256:9819b6456dd7103a16177630cb45c7e6ee6e96fac69b2ac47a327063c688d342
```

`pull` is much more easier:
```sh
Usage:  docker-app pull <repotag> [flags]

Pull an application from a registry

Options:
      --insecure   Use insecure registry, without SSL
```

* **`pull`** the image you just pushed, then pull the images of your neighbours

```sh
$ docker-app pull myneighbourhublogin/hello:0.1.0
```

## inspect

`inspect` command you already know just work directly with a remote image stored  on a registry.

* **`inspect`** your neighbours images

```sh
$ docker-app inspect myneighbour/hello:0.1.0
words 0.1.0

Services (3) Replicas Ports Image
------------ -------- ----- -----
web          1        33000 dockerdemos/lab-web
words        3              dockerdemos/lab-words
db           1              dockerdemos/lab-db
```

## install

And of course the `install` command too works the same way!

```sh
$ docker-app install myneighbour/hello:0.1.0 --name app-pulled
Creating network app-pulled_default
Creating service app-pulled_web
Creating service app-pulled_words
Creating service app-pulled_db
```

## Summary
* With push and pull commands, you can share your application like you share an image
* You can inspect it or even install it directly from a registry
