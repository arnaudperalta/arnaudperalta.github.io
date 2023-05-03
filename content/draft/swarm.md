---
title: "Never use Docker Compose again for production"
date: 2023-05-01T00:59:17+01:00
---

# Why using Docker Compose for production is bad ?

OK, that's a bit misleading, it's not THAT bad. With compose, you are defining your services with a simple YAML file and you can pull images from your registry.
So, with a good integration and a correct deploy script, you can have a good experience with Docker Compose on production.

When I say, it's bad, it's because you are one command away from using Docker Swarm, a simple container orchestrator, defined for production use.

After typing `docker swarm init` in your production server, you will get everything Docker Engine has to offer (so every Docker Compose features you like), plus, a network abstraction in case you are using multiple nodes, a secret mechanism to store safely confidentials informations (DB credentials), rolling updates, replicas ...

Swarm is known to be used on a cluster who supports multiples nodes (multiples Docker Engines), but I'll show you that a mono-node Swarm is far better than a Compose without going too much in depths.

# Small example with an average architecture

In case you are actually deploying with Docker Compose, you can keep your actual docker-compose.yml file.
But you need to change some aspects like the network and volumes.

Here is an example of a docker-compose.yml file you can deploy with Docker Compose:

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

Without secret mechanism, we need to expose credentials in plain text during our deployment (either in compose file, .env file or inside the image).
We can't scale this architecture, every services will run on a unique replica on one server.

With few changes coming from new concepts introduced by Swarm, we can get this kind of file who describes a Docker Stack:
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
        uid: "0"

  web:
    image: registry/web-$ENV:$CI_COMMIT_SHORT_SHA
    deploy:
      replicas: 2
      update_config:
        parallelism: 1
      restart_policy:
        condition: on-failure
    ports:
      - target: 8080
        published: 80

secrets:
  database_initialization:
    file: db_init.sh

volumes:
  db-data:
```

Yes, it's a bit longer to write, but you only write it once for a better deployment.

To deploy this stack, we will need to run `docker stack deploy -c <this-file>` on a Swarm node.



# Making a good integration with Swarm

If we imagine with a possible Docker Compose setup in a CI job, we could have something like:
- Retrieving SSH private key
- Creating a folder for the specific deploy
- Copying docker-compose.yml file and .env files on the server
- `docker compose pull -> up`

What I don't like with this setup, it is we are doing mutations on the target, and a lot of errors can happens because they are mutations on the filesystem on the filesystem (What if the deploy folder moved ?).

With Docker Swarm, you are only deploying a YAML file who follows the [Compose Spec](https://compose-spec.io/).
Here is an exemple with a Gitlab CI job where we deploy a stack (a set of services similar to the previous Docker Compose) on the production environment:

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
    - docker stack deploy -c docker-compose.${ENV_TO_DEPLOY}.yml app

swarm_deploy_production:
  extends: .swarm_deploy
  variables:
    SSH_USER: "me"
    SSH_HOST: "my_ssh_address"
    ENV: "production"
```

And its done, we don't need to copy anything. We changed the targeted docker daemon by changing the DOCKER_HOST variable and by using the ssh protocol.
In the before_script section we copied our SSH private key to permits Docker to use it while communicating to the daemon.

In a pipeline, we can imagine some image build stages done with [kaniko](https://github.com/GoogleContainerTools/kaniko) who whill pushes to the new registry.

# Managing sensibles datas
 
No more weird image configuration done with volumes. Now that you have a container orchastrator, you will need to include every non-sensible config in your Docker images.
For the rest, you can use the [CLI](https://docs.docker.com/engine/swarm/secrets/) `docker secret`

# You are scaling ? no worries, Swarm will do the job