# Exercise - Running and Sharing our Docker Application

> **Time**: Approximately 10 minutes
>
> **Difficulty**: Easy

## Table of Contents

1. [Deploying the Docker App](#deploying-the-docker-app)
1. [Viewing Installed Applications](#viewing-installed-applications)
1. [Uninstalling the Docker App](#uninstalling-the-docker-app)


## Exercise Objectives

By the end of this exercise, you will have:

- Deployed the Docker App using the locally-sourced app bundle
- Learned how to see existing app installations
- Removed an app installation
- Tagged and pushed the Docker App to Docker Hub
- Uninstalled a Docker App


## Deploying the Docker App

There are two different ways Docker App can locate an application bundle

- **Locally** - Docker App will use a merged or split application bundle definition found on your local filesystem
- **Remote** - Docker App will pull the application bundle from Docker Hub (or another registry)

For our first deployment, we will simply use the locally available app definitions.

1. Let's first look at the `docker app install` options. Run `docker app install --help` to see all options.

    <details>
      <summary>Full console output</summary>

    ```console
    $ docker app install --help

    Usage:  docker app install [APP_NAME] [--name INSTALLATION_NAME] [--target-context TARGET_CONTEXT] [OPTIONS]

    Install an application.
    By default, the application definition in the current directory will be
    installed. The APP_NAME can also be:
    - a path to a Docker Application definition (.dockerapp) or a CNAB bundle.json
    - a registry Application Package reference

    Aliases:
      install, deploy

    Examples:
    $ docker app install myapp.dockerapp --name myinstallation --target-context=mycontext
    $ docker app install myrepo/myapp:mytag --name myinstallation --target-context=mycontext
    $ docker app install bundle.json --name myinstallation --credential-set=mycredentials.yml

    Options:
          --credential-set stringArray    Use a YAML file containing a credential set or a credential set
                                          present in the credential store
          --insecure-registries strings   Use HTTP instead of HTTPS when pulling from/pushing to those registries
          --kubernetes-namespace string   Kubernetes namespace to install into (default "default")
          --name string                   Installation name (defaults to application name)
          --orchestrator string           Orchestrator to install on (swarm, kubernetes)
          --parameters-file stringArray   Override parameters file
          --pull                          Pull the bundle
      -s, --set stringArray               Override parameter value
          --target-context string         Context on which the application is installed (default: <current-context>)
          --with-registry-auth            Sends registry auth
    ```
    </details>

    You'll see that we need to specify the name for our application and can set the `target-context`. Remember when we talked about Docker contexts? This will be where we use that. This allows us to run the commands in the dev instance, but have the deploy actually _happen_ on our Swarm cluster. Cool, huh?

2. Let's deploy the application bundle using the `docker app install` command. Specify the name of your app as `voting-app`

    <details>
      <summary>Solution/Full Output</summary>

      ```console
      $ docker app install voting-app --target-context swarm
      Creating network front-tier
      Creating network back-tier
      Creating service voting-app_vote
      Creating service voting-app_redis
      Creating service voting-app_db
      Creating service voting-app_worker
      Creating service voting-app_result
      Application "voting-app" installed on context "swarm"
      ```
    </details>

    :tada: The application stack is now deployed! Hooray!


## Viewing Installed Applications

Another handy tool is the `docker app ls` command, which allows us to see all currently installed applications.

1. In your dev instance, run the `docker app ls` command. 

    <details>
      <summary>Full output</summary>

      ```console
      $ docker app ls
      INSTALLATION APPLICATION LAST ACTION RESULT CREATED MODIFIED REFERENCE
      ```
    </details>

    Huh... nothing showed up. Why?

2. If we look at help for `docker app ls`, we'll see an option for `--target-context` again. When we first ran the `ls` command, it asked Docker for the app installations on our current machine, which has none. Go ahead and specify the `--target-context` and see if you get a better result. As a note, you can also set the `DOCKER_TARGET_CONTEXT` env variable and Docker App will use that in all commands.

    <details>
      <summary>Full output</summary>
      
    ```console
    $ docker app ls --target-context swarm
    INSTALLATION APPLICATION        LAST ACTION RESULT  CREATED   MODIFIED  REFERENCE
    voting-app   voting-app (0.1.0) install     success 2 minutes 2 minutes
    ```
    </details>


## Uninstalling the Docker App

Let's practice uninstalling our deployed application.

1. Again from our dev instance, let's look at the full options for `docker app uninstall`.

    <details>
      <summary>Full output</summary>
      
    ```console
    $ docker app uninstall --help
    Usage:  docker app uninstall INSTALLATION_NAME [--target-context TARGET_CONTEXT] [OPTIONS]

    Uninstall an application

    Examples:
    $ docker app uninstall myinstallation --target-context=mycontext

    Options:
          --credential-set stringArray   Use a YAML file containing a credential set or a credential set
                                        present in the credential store
          --force                        Force removal of installation
          --target-context string        Context on which the application is installed (default: <current-context>)
          --with-registry-auth           Sends registry auth
    ```
    </details>

    We'll see that we need to specify the `INSTALLATION_NAME` and the `target-context`.

2. Use the `docker app uninstall` command to remove the application.

    <details>
      <summary>Solution/Full Output</summary>
    
    ```console
    $ docker app uninstall voting-app --target-context swarm
    Removing service voting-app_db
    Removing service voting-app_redis
    Removing service voting-app_result
    Removing service voting-app_vote
    Removing service voting-app_worker
    Removing network back-tier
    Removing network front-tier
    Application "voting-app" uninstalled on context "swarm"
    ```
    </details>

    :tada: It's gone now!

3. To verify, you can run `docker app ls --target-context swarm` and validate that it's gone.

    <details>
      <summary>Full output</summary>
    
    ```console
    $ docker app ls --target-context swarm
    INSTALLATION APPLICATION LAST ACTION RESULT CREATED MODIFIED REFERENCE
    ```
    </details>


## Pushing our Docker App

Now that we've learned how to deploy a locally-sourced application, let's push our app to Docker Hub and deploy from there!

1. Let's first take a look at the various options and flags on the `docker app push` command.

    <details>
      <summary>Full output</summary>
    
    ```console
    $ docker app push --help

    Usage:  docker app push [APP_NAME] --tag TARGET_REFERENCE [OPTIONS]

    Push an application package to a registry

    Examples:
    $ docker app push myapp --tag myrepo/myapp:mytag

    Options:
          --insecure-registries strings   Use HTTP instead of HTTPS when pulling from/pushing to those registries
          --platform strings              For multi-arch service images, only push the specified platforms
      -t, --tag string                    Target registry reference (default: <name>:<version> from metadata)
    ```
    </details>

2. When pushing, we will tag the application to include your Docker Hub account (`<your-hub-username>/voting-app.dockerapp`) and specify the tag as the current version (which defaulted to `0.1.0`). While not required, we do encourage you to keep the `.dockerapp` suffix on the image name to help make it more obvious that it is a repo containing a Docker App bundle.

    <details>
      <summary>Solution/Full Output</summary>
    
    ```console
    $ docker app push --tag mikesir87/voting-app.dockerapp:0.1.0
    docker.io/mikesir87/voting-app.dockerapp:0.1.0-invoc
    mikesir87/examplevotingapp_vote
    sha256:a0d4d29d...: Skip (already present)
    redis:alpine
    sha256:ef67270b...: Skip (already present)
    postgres:9.4
    sha256:094e3a9e...: Skip (already present)
    dockersamples/examplevotingapp_worker
    sha256:55753a7b...: Skip (already present)
    mikesir87/examplevotingapp_result
    sha256:69198c25...: Skip (already present)
    WARN[0003] reference for unknown type: application/vnd.cnab.config.v1+json
    Successfully pushed bundle to docker.io/mikesir87/voting-app.dockerapp:0.1.0. Digest is sha256:ca706ede7e387173cf28b20e65336299873c072deb422c35a0fd57379b46932e.
    ```
    </details>

    The output may look slightly different, as we had already previously pushed the app before capturing the output to display above. In addition, you'll see a different sha256, which can be used for addressing apps too. However, you should see a "Successfully pushed bundle" message at the end.

3. Now, let's deploy our remotely-pushed application bundle to the cluster. All we have to do is specify the remote application as the application name and Docker App will fetch the image and then use the bundled name as the _actual_ name of the installed app (which can be overridden by using the `--name` option). Don't forget to specify the `target-context` too!

    <details>
      <summary>Solution/Full Output</summary>
    
    ```console
    $ docker app install mikesir87/voting-app.dockerapp:0.1.0 --target-context swarm
    Creating network back-tier
    Creating network front-tier
    Creating service voting-app_db
    Creating service voting-app_worker
    Creating service voting-app_result
    Creating service voting-app_vote
    Creating service voting-app_redis
    Application "voting-app" installed on context "swarm"
    ```
    </details>

    And that's it! You can go onto either of the Swarm nodes and use the port badges to open the app (may take a few seconds for it pull the images and start the containers).

5. Once you've validated that it works, go ahead and uninstall the application so we have a clean slate for our next exercise.

    <details>
      <summary>Output</summary>
    
    ```console
    $ docker app uninstall voting-app --target-context swarm
    Application "voting-app" uninstalled on context "swarm"
    ```
    </details>
