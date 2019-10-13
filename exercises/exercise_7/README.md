# Exercise 7 - Advanced - Adding a Reverse Proxy (Traefik) to the Application

> **Time**: Approximately 15 minutes
>
> **Difficulty**: Medium

## Table of Contents

1. [Why a reverse proxy?](#why-a-reverse-proxy)
2. [Defining the proxy service](#defining-the-proxy-service)
3. [Updating the Vote and Results services](#updating-the-vote-and-results-services)
4. [Extra Practice - Parameterizing the Routing Rules](#extra-practice-parameterizing-the-routing-rules)

## Exercise Objectives

By the end of this exercise, you will have:

- Added a reverse proxy to the application stack
- Configured the vote and results apps to use path-based routing


## Why a reverse proxy?

As you may have already experienced, only a single container/service can listen to a given port on a cluster. But, we frequently want to have multiple webapps in our stack. Requiring users to remember all of the non-standard ports would be a disaster. A reverse proxy allows us to expose only **one** container to the standard 80 and 443 ports, where it then forwards requests to the appropriate container using container-to-container networking.

We will be updating our application stack to allow all requests to go through port 80 (the standard HTTP port) and then use path-based routing to each of the containers. The rules we will use are:

- `/*` - will be forward to the vote service
- `/results/*` - will be forwarded to the results service

This will allow the default requests to go to the `vote` service, but then any request to `/results` will be handled by the results service.

**Note:** In many environments, you may want to use host-based routing for different applications. But, we only have one domain available for us to use on PWD, so have to resort to path-based routing.


## Defining the proxy service

The first thing we need to do is add the proxy service. We will use [Traefik](https://traefik.io), a reverse proxy that easily integrates into container orchestration platforms. We will use the `traefik` image and specify a command that tells it to run in both Docker and Swarm modes, where it will watch for changes to services. As services come and go, Traefik will see the events (using the `docker.sock`) and update its config using the labels on the services.

1. In order to send traffic between the proxy and the webapps, they need to be on the same network. To accomplish this, create a new network called `proxy`.

    ```yaml
    networks:
      proxy:
        name: proxy
    ```

    Note that we are explicitly naming the network to help with the proxy configuration (which we'll see in step #3).

2. Now, it's time to add the `proxy` service. Notice this is exposing port 80, mounting the `docker.sock`, and on the `proxy` network we just created. Since it needs to listen to service events, it needs to be running on a manager.

    ```yaml
    services:
      proxy:
        image: traefik
        command:
          - '--providers.docker=true'
          - '--api.insecure=true'
          - '--entrypoints.web.address=:80'
          - '--accesslog=true'
          - '--providers.docker.swarmMode=true'
        ports:
          - 80:80
          - 8080:8080
        networks:
          - proxy
        volumes:
          - /var/run/docker.sock:/var/run/docker.sock
        deploy:
          placement:
            constraints: [node.role == manager]
    ```

    **NOTE:** We're adding the `--api` flag to enable a dashboard that is exposed on port 8080. However, this also enables a full read/write API on Traefik, which allows routing config to be changed. You would want to be secure this endpoint if you enable it in a production environment.



## Updating the Vote and Results services

With the Traefik service added, we just need to add labels to the `vote` and `results` services ([full docs found here](https://docs.traefik.io/configuration/backends/docker/#labels-overriding-default-behavior)). The labels we will define on each service are:

- `traefik.backend` - a unique name for this service
- `traefik.frontend.rule` - a [matcher](https://docs.traefik.io/basics/#matchers) that serves as a rule to be evaluated to determine when Traefik should send HTTP traffic to this service. The `PathPrefixStrip` matcher that we will use will match based on path, but then strip the path before forwarding the request (since our apps are expecting to be at the URL root)
- `traefik.port` - the container port for the service that Traefik should send the request to (defaults to port 80)
- `traefik.docker.network` - since our services are on multiple networks, we explicitly tell Traefik which network to use when forwarding the request. If it picks the wrong network (and one it's not on), the packet would be dropped and the request would eventually time out. One caveat is that the network name _must_ match exactly, which is why we explicitly named the network. If we didn't, the network name would have the stack name appended to it (like `voting-app_proxy`).

1. Add the `proxy` network (don't remove the existing networks) and the following `labels` to the `vote` service.

```yaml
services:
  vote:
    networks:
    - proxy
    deploy:
      labels:
        - 'traefik.http.routers.vote.rule=PathPrefix(`/`)'
        - "traefik.http.routers.vote.service=vote"
        - "traefik.docker.network=proxy"
        - "traefik.enable=true"
        - "traefik.http.services.vote.loadbalancer.server.port=80"
```

2. Add the `proxy` network (don't remove the existing networks) and the following labels to the `results` service.

```yaml
services:
  results:
    networks:
      - proxy
    deploy:
      labels:
        - 'traefik.http.routers.results.rule=PathPrefix(`/results`)'
        - "traefik.http.middlewares.results-prefix.stripprefix.prefixes=/results"
        - "traefik.http.routers.results.middlewares=results-prefix@docker"
        - "traefik.http.routers.results.service=results"
        - "traefik.docker.network=proxy"
        - "traefik.enable=true"
        - "traefik.http.services.results.loadbalancer.server.port=80"
```

3. Remove the `port` mappings from both the `vote` and `results` services, as we will now be accessing them through the proxy now!

4. Deploy the updated stack to your swarm cluster! You'll see the port badges for 5000 and 5001 disappear. Once the stack is up, click on the `80` badge. You should see the voting application. If you update your URL to point to `/results/`, you should see the results app.



## Extra Practice - Parameterizing the Routing Rules

As an extra bonus, see if you can update the routing rules (the `traefik.frontend.rule` labels) to be provided through parameters. With that in place, you can ship this application to anyone and they can adjust the rules to use a different path, a host, or something completely different!
