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

- Deployed the Docker App using the built app image
- Learned how to see existing app installations
- Removed an app installation
- Managed your local Application images
- Pushed the Docker App image to Docker Hub


## Run the Docker App

There are two different ways Docker App can locate an application:

- **Locally** - Docker App will use a built application 
- **Remote** - Docker App will pull the application image from Docker Hub (or another registry)

For our first deployment, we will simply use the locally available app images.

1. Let's first look at the `docker app run` options. Run `docker app run --help` to see all options.

    <details>
      <summary>Full console output</summary>

    ```console
    $ docker app run --help

    Usage:  docker app run [OPTIONS] [APP_IMAGE]

    Run an application based on a docker app image.

    Aliases:
      run, deploy

    Examples:
    $ docker app run --name myinstallation --target-context=mycontext myrepo/myapp:mytag

    Options:
          --credential stringArray        Add a single credential, additive ontop of any --credential-set used
          --credential-set stringArray    Use a YAML file containing a credential set or a credential set present in the credential store
          --name string                   Assign a name to the installation
          --namespace string              Kubernetes namespace to install into (default "default")
          --orchestrator string           Orchestrator to install on (swarm, kubernetes)
          --parameters-file stringArray   Override parameters file
      -s, --set stringArray               Override parameter value
          --target-context string         Context on which the application is installed (default: <current-context>)
          --with-registry-auth            Sends registry auth
    ```
    </details>

    You can specify the name for our application (otherwise a name will be generated) and can set the `target-context`. Remember when we talked about Docker contexts? This will be where we use that. This allows us to run the commands in the dev instance, but have the deploy actually _happen_ on our Swarm cluster. Cool, huh?

2. Let's run the application using the `docker app run` command. Specify the name of your app as `voting-app`

    <details>
      <summary>Solution/Full Output</summary>

      ```console
      $ docker app run username/voting-app:0.1.0 --name voting-app --target-context swarm
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


## Viewing running Applications

Another handy tool is the `docker app ls` command, which allows us to see all currently running applications.

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


## Removing the Docker App

Let's practice removing our running application.

1. Again from our dev instance, let's look at the full options for `docker app rm`.

    <details>
      <summary>Full output</summary>
      
    ```console
    $ docker app rm --help
    Usage:  docker app rm INSTALLATION_NAME [--target-context TARGET_CONTEXT] [OPTIONS]

    Remove an application

    Examples:
    $ docker app rm myinstallation --target-context=mycontext

    Options:
          --credential-set stringArray   Use a YAML file containing a credential set or a credential set
                                        present in the credential store
          --force                        Force removal of installation
          --target-context string        Context on which the application is installed (default: <current-context>)
          --with-registry-auth           Sends registry auth
    ```
    </details>

    We'll see that we need to specify the `INSTALLATION_NAME` and the `target-context`.

2. Use the `docker app rm` command to remove the application.

    <details>
      <summary>Solution/Full Output</summary>
    
    ```console
    $ docker app rm voting-app --target-context swarm
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

## Manage your local Docker Application images

1. Now that we know how to manage running applications, let's focus on local application images.
  Check the `docker app image` manage command with `--help` flag:

    <details>
      <summary>Solution/Full Output</summary>
    
    ```console
    $ docker app image --help
    Usage:	docker app image COMMAND

    Manage application images

    Commands:
      ls          List application images
      rm          Remove an application image
      tag         Create a new tag from an application image

    Run 'docker app image COMMAND --help' for more information on a command.
    ```
    </details>

2. List all your local application image using `docker app image ls`.

    <details>
      <summary>Solution/Full Output</summary>
    
    ```console
    $ docker app image ls
    APP IMAGE                   APP NAME
    username/voting-app:v0.1.0  voting-app
    ```
    </details>

3. Let's build our app again without tagging it
    <details>
      <summary>Solution/Full Output</summary>
    
    ```console
    $ docker app build voting-app.dockerapp
    [+] Building 0.0s (6/6) FINISHED                                                                           
    => CACHED [internal] load remote build context                                                       0.0s
    => CACHED copy /context /                                                                            0.0s
    => [internal] load metadata for docker.io/docker/cnab-app-base:v0.8.0-222-gc9b862782a                0.0s
    => [1/2] FROM docker.io/docker/cnab-app-base:v0.8.0-222-gc9b862782a                                  0.0s
    => CACHED [2/2] COPY . .                                                                             0.0s
    => exporting to image                                                                                0.0s
    => => exporting layers                                                                               0.0s
    => => writing image sha256:89ed7688293f72d01706c501d7657e26c3097b33ae4d9cd8ad0853bbe813923b          0.0s
    Successfully built service images
    Successfully build 2f519fef648813aad582c87a0aeeab7cda616447cc7356375ca634650b0ed14b
    ```
    </details>

4. List your application images again. Application images built without a tag will be shown using their id. **NOTE**: All the docker app commands using an Application image name (`image rm`, `image tag`, `run`, `inspect`) can use an Application image ID instead.

    <details>
      <summary>Solution/Full Output</summary>
    
    ```console
    $ docker app image ls
    APP IMAGE                                                        APP NAME
    2f519fef648813aad582c87a0aeeab7cda616447cc7356375ca634650b0ed14b voting-app
    username/voting-app:v0.1.0                                       voting-app
    ```
    </details>

5. Tag an application image using `docker app image tag` command. You can re-tag an image as much as you want, as you would do with `docker image tag` command. This is usefull if you need to name an unnamed built application before pushing it, or you want to push your application image to a different registry. Let's tag the unnamed pre-built application image. Then list again your application images to check the tags. **NOTE**: default tag is `latest`.

    <details>
      <summary>Solution/Full Output</summary>
    
    ```console
    $ docker app image tag 2f519fef648813aad582c87a0aeeab7cda616447cc7356375ca634650b0ed14b myregistry/username/voting-app
    $ docker app image ls
    APP IMAGE                                                        APP NAME
    2f519fef648813aad582c87a0aeeab7cda616447cc7356375ca634650b0ed14b voting-app
    username/voting-app:v0.1.0                                       voting-app
    myregistry/username/voting-app:latest                            voting-app
    ```
    </details>

6. Remove an application image using `docker app image rm`. You can now remove your unnamed application as it has been tagged and list application images to check it has been cleaned.
    <details>
      <summary>Solution/Full Output</summary>
    
    ```console
    $ docker app image rm 2f519fef648813aad582c87a0aeeab7cda616447cc7356375ca634650b0ed14b
    Deleted: 2f519fef648813aad582c87a0aeeab7cda616447cc7356375ca634650b0ed14b
    $ docker app image ls
    APP IMAGE                             APP NAME
    username/voting-app:v0.1.0            voting-app
    myregistry/username/voting-app:latest voting-app
    ```
    </details>

## Pushing our Docker Application image

Now that we've learned how to run a locally built application, let's push our app to Docker Hub and run it from there!

1. Let's first take a look at the `docker app push` command. 

    <details>
      <summary>Full output</summary>
    
    ```console
    Usage:  docker app push [APP_NAME] --tag TARGET_REFERENCE [OPTIONS]

    Push an application package to a registry

    Examples:
    $ docker app push myapp --tag myrepo/myapp:mytag

    Options:
          --all-platforms      If present, push all platforms
          --platform strings   For multi-arch service images, push the specified platforms (default [linux/amd64])
      -t, --tag string         Target registry reference (default: <name>:<version> from metadata)
    ```
    </details>

2. Before pushing, make sure you tag the application image to include your Docker Hub account (`<your-hub-username>/voting-app`) and specify the tag as the current version (which defaulted to `latest`).

    <details>
      <summary>Solution/Full Output</summary>
    
    ```console
    $ docker app push username/voting-app:0.1.0
    docker.io/username/voting-app1:0.1.0-invoc
    postgres:9.4
    application/vnd.docker.distribution.manifest.list.v2+json [1/1] (sha256:ded83722...)
    redis:alpine
    application/vnd.docker.distribution.manifest.list.v2+json [1/1] (sha256:3ffbafc4...)
    mikesir87/examplevotingapp_result
    application/vnd.docker.distribution.manifest.v2+json [10/10] (sha256:69198c25...)
    mikesir87/examplevotingapp_vote
    application/vnd.docker.distribution.manifest.v2+json [8/8] (sha256:a0d4d29d...)
    dockersamples/examplevotingapp_worker
    application/vnd.docker.distribution.manifest.v2+json [10/10] (sha256:55753a7b...)
    Successfully pushed bundle to docker.io/username/voting-app1:0.1.0. Digest is sha256:ec6c0449615167865322e34563f95d495f4fcd106f9780ed07
    271345107a080b.
    ```
    </details>

    The output may look slightly different, as we had already previously pushed the app image before capturing the output to display above. In addition, you'll see a different sha256, which can be used for addressing apps too. However, you should see a "Successfully pushed" message at the end.

3. Now, let's run our remotely-pushed application image to the cluster. All we have to do is specify the remote application image and Docker App will fetch the image and then use it for deployment. Don't forget to specify the `target-context` too!

    <details>
      <summary>Solution/Full Output</summary>
    
    ```console
    $ docker app run username/voting-app:0.1.0 --name voting-app --target-context swarm
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

5. Once you've validated that it works, go ahead and remove the application so we have a clean slate for our next exercise.

    <details>
      <summary>Output</summary>
    
    ```console
    $ docker app rm voting-app --target-context swarm
    Application "voting-app" removed on context "swarm"
    ```
    </details>
