# Getting started

This repository is a sample application for users following the getting started guide at https://docs.docker.com/get-started/.

The application is based on the application from the getting started tutorial at https://github.com/docker/getting-started

# Docker Workshop

## Part 1: Containerize an application

1. Buid the app's image
- Create `Dockerfile` inside this directory, same location as `package.json`

```Dockerfile
# syntax=docker/dockerfile:1

FROM node:lts-alpine
WORKDIR /app
COPY . .
RUN yarn install --production
CMD ["node", "src/index.js"]
EXPOSE 3000
```

- Build image
```bash
docker build -t getting-started .
# -t (--tag): flag tags image
```

2. Start app container
- Run container

```bash
docker run -d -p 127.0.0.1:3000:3000 getting-started
# -d (--detach): runs container in background
# -p (--publish): creates port mapping between host and container
```

- Verify the app at http://localhost:3000


## Part 2: Update the application

1. Update source code

- In the `src/statci/js/app.js` file, update

```js
- <p className="text-center">No items yet! Add one above!</p>
+ <p className="text-center">You have no todo items yet! Add one above!</p>
```

- Build the update version of the image

```bash
docker build -t getting-started .
```

- Start new container using the updated code => ERROR!

```bash
docker run -dp 127.0.0.1:3000:3000 getting-started
```

It's because we aren't able to start new container while the old one is still running as it's already using the host's port 3000.

2. Remove old container

We need to stop a container before removing it.

- Get ID of that container:

```bash
docker ps
```

- Stop that container:

```bash
docker stop <container-id>
```

- Remove that container:

```bash
docker rm <container-id>
```

3. Start the updated app container

```bash
docker run -dp 127.0.0.1:3000:3000 getting-started
```

Now it works, refresh the browser on http://localhost:3000/ and see the updated text.


## Part 3: Share the application

1. Create a repository

- Create respository on Docker Hub with name `getting-started`


- Make sure login on machine

```bash
docker login
```

2. Push the image

- Try to push image => ERROR

```bash
docker push docker/getting-started
```

It's because local image isn't tagged correctly yet.

- Tag new name to local image

```bash
docker tag getting-started YOUR-USER-NAME/getting-started
```

- Then push

```bash
docker push YOUR-USER-NAME/getting-started
```

3. Run image on a new instance

- On Macbook, we need to rebuild image to be compatible with [`Play with Docker`](https://labs.play-with-docker.com/) then repush the image

```bash
docker build --platform linux/amd64 -t YOUR-USER-NAME/getting-started .
docker push YOUR-USER-NAME/getting-started
```

- On [`Play with Docker`](https://labs.play-with-docker.com/), 
select `ADD NEW INSTANCE`, then start the app

```bash
docker run -dp 0.0.0.0:3000:3000 YOUR-USER-NAME/getting-started
```

- Click to `OPEN PORT` then specify `3000`, the app will be apprear.


## Part 4: Persist the DB

