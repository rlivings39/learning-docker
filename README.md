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
