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
docker run -dp 3000:3000 getting-started
```

