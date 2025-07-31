# Learning Docker

Going through [Docker 101 tutorial](https://www.docker.com/101-tutorial/)

1. Run `docker run -dp 82:80 docker/getting-started`
2. Open http://localhost:82/
3. Prosper

## Notes

In the command `docker run -dp 82:80 docker/getting-started` `-d` means to run detached (i.e. in the background) and `-p` maps port 80 in the container to 82 on the host.

Docker uses **containers**. A **container** is a process on your machine that has been isolated from other processes. It leverages [kernel namespaces and cgroups](https://medium.com/@saschagrunert/demystifying-containers-part-i-kernel-space-2c53d6979504).

Running containers use an isolated filesystem. The custom filesystem is provided by a **container image**. The image must include everything needed to run the application - all dependencies, configuration, scripts, binaries, etc. It also contains configuration for the container like environment variables, a default command to run, and other metadata.

This filesystem isolation is like an advanced version of `chroot`.

To build an image for something you use a `Dockerfile` which is a textual script of instructions used to create the image.

A Dockerfile like

```docker
FROM node:18-alpine
WORKDIR /app
COPY . .
RUN yarn install --production
CMD ["node", "src/index.js"]
```

says to build from the Alpine Linux Node.js image, set up a workdir called app, copy the current folder to the working dir, run `yarn install --production` in the container, and to run `node src/index.js` to start the image.

To build an image from a Dockerfile run `docker build -t image-name .`

To run our test app container do

```docker
docker run --name getting_started -dp 3000:3000 getting-started
```

`docker ps` will show running containers

`docker stop container-id` will stop it

`docker rm container-id` removes it

`docker rm -f container-id` will stop and remove it

These can also be done from the Docker dashboard.

## Sharing images

To share an image you can create a repository on Docker Hub and run

```bash
docker login -u your-user-name
docker tag getting-started your-user-name/getting-started
docker push your_namespace/getting-started:tagname
```

You can now run this image on a new instance where it's never been installed.

## Persisting data

The container has its own filesystem and scratch space where files can be written/stored. Multiple instances of the same image do **not** share filesystem changes.

To run things inside of the container you can use

```bash
docker exec <container-id> cat /data.txt
```

If you use `docker run -it ...` this launches an interactive terminal in the container.

**Volumes** allow connecting filesystems inside the container back to the host filesystem. If the same host directory is mounted across container restarts, then temporary work will be preserved.

The app uses an SQLite database stored in `/etc/todos/todo.db`. We can use a **named volume** in Docker to persist the data from the container to the host.

```bash
docker volume create todo-db
```

Then you can mount the volume using the `-v` flag

```
docker run --name getting_started -dp 3000:3000  -v todo-db:/etc/todos getting-started
```

That says to make the named volume `todo-db` visible inside the container at `/etc/todos/`.

After removing and running that container again, you'll see that the data has been persisted.

```
docker volume inspect todo-db
```

will show info about the volume including where it is stored.

**Note** Named volumes and bind mounts are the main types of volumes. There are many volume driver plugins to support NFS, SFTP, NetApp, and more. Those are useful when working with multiple hosts in Swarm, Kubernetes, etc.

**Bind mounts** allow you to specify the mount point from the host. They do not populate the volume with container contents and do not support mount drivers. They're useful for storing data and to provide data into the application.

```
docker run ... -v "$(pwd):/app" node:18-alpine sh -c "yarn install && yarn run dev"
```

will mount your source code into the container and launch the dev server so it responds to changes being made to the code.

```
docker logs -f container-id
```

shows container logs so you can know what it's doing.

## Multi-container apps

**Each container should do one thing and do it well**. Separate containers allow scaling and updating systems independently, give flexibility to use different services in dev and prod, etc.

Containers support **networking**. If two containers are on the same network, they can communicate. If they're on different networks, they can't communicate.

```
docker network create todo-app
```

creates a network that can be used when launching an image
```
docker run -d \
    --network todo-app --network-alias mysql \
    -v todo-mysql-data:/var/lib/mysql \
    -e MYSQL_ROOT_PASSWORD=secret \
    -e MYSQL_DATABASE=todos \
    mysql:8.0
```

The `--network-alias` flag specifies the name which can be used to refer to the container on the network.

**Note** Managing secrets with environment variables is ok for dev work but **bad** for production. There are many secret managers in things like Docker swarm and Kubernetes. Here's a [blog post](https://blog.diogomonica.com/2017/03/27/why-you-shouldnt-use-env-variables-for-secret-data/) showing why environment variables are bad for secrets.

Docker compose allows coordinating multiple containers by defining things in a `docker-compose.yml` YAML file. You define this configuration in your repo and source control it. See the [YAML file](./docker-compose.yml) in this repo as an example.

To launch the container run

```
docker compose up -d

[+] Running 4/4
 ✔ Network learning-docker_default           Created    0.3s
 ✔ Volume "learning-docker_todo-mysql-data"  Created    0.0s
 ✔ Container learning-docker-app-1           Started    1.1s
 ✔ Container learning-docker-mysql-1         Started    1.1s
```

Note that this automatically creates a network to connect your containers together.

```
docker compose logs -f app
```
shows the logs and follows output.

```
docker compose down
```

tears everything down. Adding the `--volumes` flag to `docker compose down` will remove volumes.

## Image building best practices

```
docker scan image-name
```

scans for security vulnerabilities. Docker Hub can be configured to scan for vulnerabilities.

```
docker image history image-name --no-trunc
```

shows layers in the image. Oldest are at the bottom and newest are at the top.


Copying in just `package.json yarn.lock`, then running `yarn install`, and finally copying in everything else allows caching so that dependencies are not reinstalled/created every time.

Caching can significantly speed up builds.

A `.dockerignore` file is used to skip copying files like `node_modules`. Read more about [containerizing Node.js web apps](https://nodejs.org/en/docs/guides/nodejs-docker-webapp/).

**Multi-stage builds** allow separating build-time dependencies from runtime dependencies. Only the last stage is included in the final image. Here's a React example

```docker
FROM node:18 AS build
WORKDIR /app
COPY package* yarn.lock ./
RUN yarn install
COPY public ./public
COPY src ./src
RUN yarn run build

FROM nginx:alpine
COPY --from=build /app/build /usr/share/nginx/html
```

## Containerizing a Node.js app

https://docs.docker.com/guides/nodejs/

To create a new Docker app skeleton you can run

```
docker init
```

and answer the questions.

Using `docker compose` you can combine multiple containers together and declare dependencies between them.

`healthcheck` allows checking on the status of a service to see if it's still alive.

`secrets` allows defining secrets that are read in from files which can be reused in other parts of the container.

To set up a development environment you can use a multi stage build in your dockerfile with a `base` for the base image, `dev` for the dev setup/build, and `prod` as your production build. These stages can be referred to by name in the compose file.

Running

```
docker compose run server npm run test
```

will allow you to run your tests. You can add a `test` stage to the Dockerfile that uses a `RUN` instead of `CMD` to run tests during container build. Doing that ensures the build fails if the tests fail.

```
docker build -t node-docker-image-test --progress=plain --no-cache --target test .
```

The guide shows how to set up [CI/CD for the app](https://docs.docker.com/guides/nodejs/configure-ci-cd/) to test, build, and deploy to Docker Hub.

That introduced the idea of adding **repository variables** and **repository secrets** in GitHub that can be referenced in CI/CD.

Finally they discuss [testing locally in Kubernetes](https://docs.docker.com/guides/nodejs/deploy/).

## Managing secrets

Tools like [Mozilla SOPS](https://github.com/getsops/sops) (w/ age), Ansible/Puppet vault modes, Hashicorp vault, Vaultwarden, GitHub secrets, AWS Secrets Manager, Azure Key Vault, Google Cloud Secret Manager were all mentioned. **Secret management** is the search term.

Stub .env files with empty placeholders for the secrets.

BUT, environment variables aren't very secure. They can often be read from outside the running process, might show up in command history when launching your service, and are by default visible to all libraries you use in your service & child processes.

Tips from [GitHub](https://docs.github.com/en/get-started/learning-to-code/storing-your-secrets-safely)

* Follow the principle of least privilege. Restrict secrets to have the minimum permissions necessary.
* Protect secrets in your application. Never hardcode them in code. Use environment variables or secret management tools like GitHub secrets.
* Set expiration dates and rotate regularly
* Ensure secrets are redacted during logging
* Limit damage if a secret is exposed. Revoke immediately if you suspect something leaked.
* Monitor activity logs to unauthorized access
* Harden based on your experience

GitHub has a secret manager that integrates with actions.

## What's next

**Container orchestration** with Swarm, Kubernetes, Nomad, and ECS all solve the problem of keeping your containers alive and the system running like it should. There's a manager that gets a definition of **expected state** like "run 2 instances of my app and expose port 80". The managers watch for changes, like a container quitting, and then work to make the **actual state** match the expected state.

Cloud native computing foundation projects provide lots of projects in the cloud computing space.

## Next

- [ ] https://docs.docker.com/guides/nodejs/containerize/
- [ ] https://fastapi.tiangolo.com/deployment/docker/#a-simple-app
