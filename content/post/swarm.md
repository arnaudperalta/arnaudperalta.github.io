---
title: "Never use Docker Compose again for production"
date: 2023-05-08T00:59:17+01:00
---

## Why using Docker Compose for production is bad?

OK, that's a bit misleading, it's not THAT bad. With compose, you are defining your services with a simple YAML file and you can pull images from your registry.
So, with a good integration and a correct deploy script, you can have a good experience with Docker Compose on production.

When I say, it's bad, it's because you are one command away from using Docker Swarm, a simple container orchestrator, defined for production usage.

After typing `docker swarm init` in your production server, you will get everything Docker Engine has to offer, plus, the possibility to host your applications on a multi-node architecture without headaches, a new docker network type (overlay) as the default load balancer packed with Swarm, a more secure secret mechanism to store safely confidential informations (DB credentials), rolling updates, replicas ...

Swarm is known to be used on a cluster that supports multiples nodes (multiples Docker Engines) to create a high resilient system in case one of your node is going off.
But I'll show you that a mono-node Swarm is far better than a Compose without going too much in-depth.

## Small example with an average architecture

In case you are actually deploying with Docker Compose, you can keep your actual docker-compose.yml file.
In effect, the way Docker Compose and Swarm deployment are defined follows the [Compose Spec](https://compose-spec.io/).

Here is an example of a docker-compose.yml file you can deploy with Docker Compose (I'm using Gitlab CI variables by the way):

```yml
version: "3.9"
services:
  db:
    image: postgres:9.4
    environment:
      POSTGRES_USER: pguser
      POSTGRES_PASSWORD: pguser
      POSTGRES_DB: pgdb
    volumes:
      - db_storage:/var/lib/postgresql

  web:
    image: registry/web-$ENV:$CI_COMMIT_SHORT_SHA
    ports:
      - "80:8080"
  
volumes:
  db_storage:
```

Without a secret mechanism, we need to expose credentials in plain text during our deployment (either in compose file, .env file, or inside the image).
We can't scale this architecture, every service will run on a unique replica on one server.

With few changes coming from new concepts introduced by Swarm, we can get this kind of file that describes a Docker Stack (A stack is defined by a compose file, you can deploy multiples stack on Swarm cluster):
```yml
version: "3.9"
services:
  db:
    image: postgres:9.4
    volumes:
      - type: volume
        source: db-data
        target: /var/lib/postgresql/data
        volume:
          nocopy: true
    secrets:
      - source: db_init
        target: "/docker-entrypoint-initdb.d/db_init.sh"
        mode: 0755
        uid: '0'

  web:
    image: registry/web-$ENV:$CI_COMMIT_SHORT_SHA
    deploy:
      replicas: 2
      update_config:
        parallelism: 1
      restart_policy:
        condition: on-failure
    ports:
      - '8080:80'
    secrets:
       - db_credentials

secrets:
  db_init:
    external: true
  db_credentials:
    external: true

volumes:
  db-data:
```

Yes, it's a bit longer to write, but you only write it once for better deployment.
We defined a persistent volume for the database. The way is it done is a little bit different for Swarm, because the volume needs to be unique across the cluster for a unique replica.
In most cases, the database must have 1 replica.

The service `web` is now defined to be deployed on two replicas, and the update process will update the service 1 replica with another. With this trick, our uptime will be at 100% because at least one replica will be up during a new deployment.

We are not longer exposing credentials in plain text, here we are using a secret named `db_init` which is a file that will be copied in the `target` location. So we can imagine that the secret value is a shell script that exports all of the DB credentials.
Same for the service `web`, we are using a secret to get access to the database credentials without exposing them.

To deploy this stack, we will need to run `docker stack deploy -c <this-file>` on a Swarm node.

## Making a good integration with Swarm

If we imagine a possible Docker Compose setup in a CI job, we could have something like this:
- Retrieving SSH private key
- Creating a folder for the specific deploy
- Copying docker-compose.yml file and .env files on the server
- `docker compose pull -> up`

What I don't like with this setup, is we are doing mutations on the target, and a lot of errors can happen because they are mutations on the filesystem on the filesystem (What if the deploy folder moved ?).

Here is an example with a Gitlab CI job where we deploy a stack (a set of services similar to the previous Docker Compose) on the production environment by only sending to the Docker daemon deployments information, nothing is added or deleted on the server from the CI job:

```yml
.swarm_deploy:
  image: docker:20.10.24
  variables:
    DOCKER_HOST: "ssh://${SSH_USER}@${SSH_HOST}"
  before_script:
    - 'which ssh-agent || ( apk --update add openssh-client )'
    - eval $(ssh-agent -s)
    - ssh-add <(echo "$SSH_KEY")
    - mkdir -p ~/.ssh
    - '[[ -f /.dockerenv ]] && echo -e "Host *\n\tStrictHostKeyChecking no\n\n" > ~/.ssh/config'
  script:
    - docker login <your registry>
    - docker stack deploy -c docker-compose.yml app

swarm_deploy_production:
  extends: .swarm_deploy
  variables:
    SSH_USER: "me"
    SSH_HOST: "my_ssh_address"
    ENV: "production"
```

And it is done, we don't need to copy anything. We changed the targeted docker daemon by changing the DOCKER_HOST variable and by using the ssh protocol.
In the before_script section, we copied our SSH private key to allow Docker to use it while communicating with the daemon.

In a pipeline, we can imagine some image build stages done with [kaniko](https://github.com/GoogleContainerTools/kaniko) who will push to the new registry.

## You are scaling? no worries, Swarm will do the job

Your application is getting more and more load on it and your single node is not enough to support all of it.

As I explained before, we can set several replicas for each service in a stack.
At first, we can increase the number of replicas on our single-node cluster.
If it is not enough, we can add a new node to the cluster, see the Docker [documentation]([documentation](https://docs.docker.com/engine/swarm/swarm-tutorial/add-nodes/)).

Docker Swarm will manage alone all of the distribution of the containers between the nodes and all of the complexity is removed thanks to the `overlay` network type which makes a great abstraction between nodes.
Here is an illustration from the Docker documentation of the default network `ingress`:

![image](https://docs.docker.com/engine/swarm/images/ingress-routing-mesh.png)

Even if you increase the number of nodes in your cluster, the `ingress` network will manage the load-balance of every request from the outside.