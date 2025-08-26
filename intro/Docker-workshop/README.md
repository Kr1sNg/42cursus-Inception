# Getting started

This repository is a sample application for users following the getting started guide at https://docs.docker.com/get-started/.

The application is based on the application from the getting started tutorial at https://github.com/docker/getting-started

# Docker Workshop

## Part 1: Containerize an application

### 1. Buid the app's image
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

### 2. Start app container
- Run container

```bash
docker run -d -p 127.0.0.1:3000:3000 getting-started
# -d (--detach): runs container in background
# -p (--publish): creates port mapping between host and container
```

- Verify the app at http://localhost:3000


## Part 2: Update the application

### 1. Update source code

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

### 2. Remove old container

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

### 3. Start the updated app container

```bash
docker run -dp 127.0.0.1:3000:3000 getting-started
```

Now it works, refresh the browser on http://localhost:3000/ and see the updated text.


## Part 3: Share the application

### 1. Create a repository

- Create respository on Docker Hub with name `getting-started`

- Make sure login on machine

```bash
docker login
```

### 2. Push the image

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

### 3. Run image on a new instance

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

### 1. Container's filesystem

When a container runs, it uses the various layers form an image for its filesystem. Each container also gets its own "scratch space" to create, update or remove files. Any changes won't be seen in another container, even if they're using the same image.

### 2. Container Volumes

Volumes provide ability to connect specific filesystem paths of the container back to the host machine.
If we mount a directory in the container, changes in that directory are also seen on the host machine. If we mount that same directory across container restarts, we'd see the same files.
There're two methods of mount: volume mount (`type=volume`) and bind mount (`type=bind`).
We will discover volume mount in this chapter.

### 3. Persist the todo data

By default, the todo app stores its data in a SQLite database at `/etc/todos/todo.db` in the container's filesystem. With the database being a single file, if we can persist that file on the host and make it available to the next container, it should be able to pick up where the last one left off. By creating a volume and attaching (often called "mounting") it to the directory where we stored the data, we can persist the data.

- Create a volume and start container

```bash
docker volume create todo-db

docker rm -f <container-id>	#remove old container if existed

docker run -dp 127.0.0.1:3000:3000 --mount type=volume,src=todo-db,target=/etc/todos getting-started
```

- Verify that the data persists by open the app then add few items to todo list.

- Stop and remove the container

- Start new container and re-open the app, we should see the item still in the list.

### 4. Dive into the volume

- "Where is Docker storing my data when I use a volume?"

```bash
docker volume inspect todo-db
```

## Part 5: Use bind mounts

A bind mount is another type of mount, which lets we share a dirctory from the host machine into the container.

### 1. Quick volume type comparisons

Two types of mount use the same flag `--mount`:
- Named volume: type=volume,src=my-volume,target=/usr/local/data
- Bind mount: type=bind,src=/path/to/data,target=/usr/local/data

### 2. Trying out bind mounts

This is example of creating bind mount.

```bash
docker run -it --mount type=bind,src="$(pwd)",target=/src ubuntu bash
```

The `--mount type=bind` option tells Docker to create a bind mount, where `src` is the current working directory on the host machine (this `Docker-workshop`), and target is where that directory should appear inside the container (`/src`).

Docker starts an interactive `bash` session in the root directory of the container's filesystem.

```bash
root@ac1237fad8db:/# pwd
/
root@ac1237fad8db:/# ls
bin   dev  home  media  opt   root  sbin  srv  tmp  var
boot  etc  lib   mnt    proc  run   src   sys  usr
```

Create and modify a random file and see the change in the container's volume and on the host.

```bash
root@ac1237fad8db:/# cd src
root@ac1237fad8db:/src# ls
Dockerfile  node_modules  package.json  spec  src  yarn.lock
root@ac1237fad8db:/src# echo hello > myfile.txt
root@ac1237fad8db:/src# ls
Dockerfile  myfile.txt  node_modules  package.json  spec  src  yarn.lock
```

### 3. Development containers

Using bind mounts is common for local development setups. The advantage is that the development machine doesn’t need to have all of the build tools and environments installed. With a single docker run command, Docker pulls dependencies and tools.

a. Run app in a development container

- Make sure there's no container currently running.

- Run a development container which mounts sourcode into the container, installs all dependencies, starts `nodemon` to watch for filesystem changes.

```bash
docker run -dp 127.0.0.1:3000:3000 \
    -w /app --mount type=bind,src="$(pwd)",target=/app \
    node:18-alpine \
    sh -c "yarn install && yarn run dev"
```

`-dp 127.0.0.1:3000:3000` - same as before. Run in detached (background) mode and create a port mapping.

`-w /app` - sets the "working directory" or the current directory that the command will run from.

`--mount type=bind,src="$(pwd)",target=/app` - bind mount the current directory from the host into the `/app` directory in the container.

`node:18-alpine` - the image to use. Note that this is the base image for the app from the Dockerfile.

`sh -c "yarn install && yarn run dev"` - the command. We're starting a shell using `sh` (alpine doesn't have `bash`) and running `yarn install` to install packages and then running `yarn run dev` to start the development server. As look in the `package.json`, we can see that the `dev` script starts `nodemon`.

- Watch the logs by

```bash
docker logs -f <container-id>
```

b. Develop the app with the development container

Update the app on host machine and see the changes reflected in the container.

- In the `src/static/js/app.js` file, on line 109, change the `"Add Item"` button to simply say `"Add"`:

```js
{submitting ? 'Adding...' : 'Add'}
```

- Refresh the page http://localhost:3000/ 

- Each time we make a change and save a file, the change is reflected in the container because of the bind mount. When Nodemon detects a change, it restarts the app inside the container automatically.

- Stop container, build the new image and push:

```bash
docker stop <container-id>
docker build -t <your-id>/getting-started .
docker push <your-id>/getting-started
```

## Part 6: Multi-container apps

in this chapter, we will add MySQL to application stack.

### 1. Container networking

If we place two different containers on the same network, they can talk to each other.
There're two ways to put a container on a network:
- Assign the network when starting the container (<= showing in this chapter)
- Connect an already running container to a network.

### 2. Start MySQL

- Create the network

```bash
docker network create todo-app
```

- Start a MySQL container with a few environment variables and attach it to the network.

```bash
docker run -d \
    --network todo-app --network-alias mysql \
    -v todo-mysql-data:/var/lib/mysql \
    -e MYSQL_ROOT_PASSWORD=secret \
    -e MYSQL_DATABASE=todos \
    mysql:8.0
```

- Confirm that we have the dababase up and running, connect to the database and verify that it connects.

```bash
docker exec -it <mysql-container-id> mysql -u root -p
```

- Password is `secret`. Verify that we have `todos` databases in the list of databases then exit MySQL.

```bash
mysql> SHOW DATABASES;
mysql> exit
```

### 3. Connect to MySQL

We will take advantage of `nicolaka/netshoot` container, which ships with tools that are useful for troubleshooting or debugging networking issues.

- Start new container using `nicolaka/netshoot` image, make sure it connect to the same network `todo-app`

```bash
docker run -it --network todo-app nicolaka/netshoot
```

- Inside this container, we're going to use the dig command, which is a useful DNS tool. We're going to look up the IP address for the hostname `mysql`.

```bash
dig mysql
```

- In the "ANSWER SECTION", we can see an `A` record for `mysql` that resolves to `172.21.0.2` (or different value). While `mysql` isn't normally a valid hostname, Docker was able to resolve it to the IP address of the container that had that network alias (as we've used the `--network-alias` in step 2.)

What this means is that the app only simply needs to connect to a host named `mysql` and it'll talk to the database MySQL.

### 4. Run the app service with MySQL

There're few environment variables to specify MySQL connection settings:

* `MYSQL_HOST` - the hostname for the running MySQL server.
* `MYSQL_USER` - the username to use for the connection.
* `MYSQL_PASSWORD` - the password to use for the connection.
* `MYSQL_DB` - the database to use once connected.

Now, start the dev-ready container.

- Specify each of the previous environment variables, as well as connect the container to the app network.

```bash
docker run -dp 127.0.0.1:3000:3000 \
  -w /app -v "$(pwd):/app" \
  --network todo-app \
  -e MYSQL_HOST=mysql \
  -e MYSQL_USER=root \
  -e MYSQL_PASSWORD=secret \
  -e MYSQL_DB=todos \
  node:18-alpine \
  sh -c "yarn install && yarn run dev"
```

- If we look at the logs for the container (`docker logs -f <container-id>`), we should see a message similar to the following, which indicates it's using the mysql database.

```bash
docker logs -f <container-id>
nodemon src/index.js
[nodemon] 2.0.20
[nodemon] to restart at any time, enter `rs`
[nodemon] watching dir(s): *.*
[nodemon] starting `node src/index.js`
Connected to mysql db at host mysql
Listening on port 3000
```

- Open the app in the browser and add fews items to the todo list.

- Connect to mysql database and prove that the items are being written. Password is `secret`.

```bash
docker exec -it <mysql-container-id> mysql -p todos
```

Then in mysql shell, show all items

```sql
select * from todo_items;
```

## Part 7: Use Docker Compose

Docker Compose is a tool that helps we define and share multi-container applications. With Compose, we can create a YAML file to define the services and with a single command, we can spin everything up or tear it all down.

The big advantage of using Compose is that we can define the app 
stack in a file, keep it at the root of project repository, and easily enable someone else to contribute to this project. Someone would only need to clone this repository and start the app using Compose.

### 1. Create the Compose file

In file directory of this project, create a file named `compose.yaml`.

### 2. Define the app service

a. Define the app service

In Part 6.4, we used the command to start the app service:

```bash
docker run -dp 127.0.0.1:3000:3000 \
  -w /app -v "$(pwd):/app" \
  --network todo-app \
  -e MYSQL_HOST=mysql \
  -e MYSQL_USER=root \
  -e MYSQL_PASSWORD=secret \
  -e MYSQL_DB=todos \
  node:18-alpine \
  sh -c "yarn install && yarn run dev"
```

Now, we define this service in the `compose.yaml` file.

- Open `compose.yaml` in a code editor, and start by defining the name and image of the first service (or container) we want to run as part of application. The name will automatically become a network alias, which will be useful when defining the MySQL service.

```yaml
services:
  app:
    image: node:18-alpine
```

- Typically, we will see `command` close to the `image` definition, although there is no requirement on ordering. Add the `command` to `compose.yaml file`

```yaml
services:
  app:
    image: node:18-alpine
    command: sh -c "yarn install && yarn run dev"
```

- Now migrate the `-p 127.0.0.1:3000:3000` part of the command by defining the `ports` for the service

```yaml
services:
  app:
    image: node:18-alpine
    command: sh -c "yarn install && yarn run dev"
    ports:
      - 127.0.0.1:3000:3000
```

- Next, migrate both the working directory `-w /app` and the volume mapping `-v "$(pwd):/app"` by using the `working_dir` and `volumes` definitions (in container)

```yaml
services:
  app:
    image: node:18-alpine
    command: sh -c "yarn install && yarn run dev"
    ports:
      - 127.0.0.1:3000:3000
    working_dir: /app
    volumes:
      - ./:/app
```

- Finally, we need to migrate the environment variable definitions using the `environment` key

```yaml
services:
  app:
    image: node:18-alpine
    command: sh -c "yarn install && yarn run dev"
    ports:
      - 127.0.0.1:3000:3000
    working_dir: /app
    volumes:
      - ./:/app
    environment:
      MYSQL_HOST: mysql
      MYSQL_USER: root
      MYSQL_PASSWORD: secret
      MYSQL_DB: todos
```

b. Define the MySQL service

In part 6.2, we used this command to start MySQL service:

```bash
docker run -d \
  --network todo-app --network-alias mysql \
  -v todo-mysql-data:/var/lib/mysql \
  -e MYSQL_ROOT_PASSWORD=secret \
  -e MYSQL_DATABASE=todos \
  mysql:8.0
```

Now, we define it into `compose.yaml` file.

- First define the new service and name it `mysql` so it automatically gets the network alias. Also specify the image to use as well.

```yaml
services:
  app:
    # The app service definition
  mysql:
    image: mysql:8.0
```

- Next, define the volume mapping. When we ran the container with `docker run`, Docker created the named volume automatically. However, that doesn't happen when running with Compose. We need to define the volume in the top-level `volumes:` section and then specify the mountpoint in the service config. By simply providing only the volume name, the default options are used.

```yaml
services:
  app:
    # The app service definition
  mysql:
    image: mysql:8.0
    volumes:
      - todo-mysql-data:/var/lib/mysql

volumes:
  todo-mysql-data:
```

- Finally, specify the environment variables

```yaml
services:
  app:
    # The app service definition
  mysql:
    image: mysql:8.0
    volumes:
      - todo-mysql-data:/var/lib/mysql
    environment:
      MYSQL_ROOT_PASSWORD: secret
      MYSQL_DATABASE: todos

volumes:
  todo-mysql-data:
```

### 3. Run the application stack

- Make sure no other copies of containers are running.

```bash
docker ps
docker stop <container-id>
docker rm -f <container-id>
```

- Start up the app stack using compose file. Add the `-d` flag to run everything in the background.

```bash
docker compose up -d
```

- Look at the logs to check all connected.

```
docker compose logs -f
docker compose logs -f app
```

- Open app in browser on http://localhost:3000/

### 4. Tear it all down

When we're ready to tear it all down, simply run this command to stop and remove the containers

```bash
docker compose down
docker compose down --volumes #to clear also the volumes
```

## Part 8: Image-building best practice

### 1. Image layering

- To see the layers in the image we created

```bash
docker image history getting-started
```

In the output, each of the lines represents a layer in the image. The display here shows the base at the bottom with the newest layer at the top. Using this, we can also quickly see the size of each layer, helping diagnose large images.


### 2. Layer caching

Once a layer changes, all downstream layers have to be recreated as well.

We can easily see that each command in the `Dockerfile` becomes a new layer in the image. When we make a change to the image, the yarn dependencies have to be reinstalled, and it doesn't make sense to ship around the same dependencies every time we build. So we should update the `Dockerfile`

- Update the `Dockerfile` to copy in the `package.json` first, install dependencies, then copy everyting else in

```Dockerfile
# syntax=docker/dockerfile:1
FROM node:lts-alpine
WORKDIR /app
COPY package.json yarn.lock ./
RUN yarn install --production
COPY . .
CMD ["node", "src/index.js"]
```

- Build a new image using updated `Dockerfile`

```bash
docker build -t <login-id>/getting-started .
```

- In file `src/static/index.html`, change `<title>` become "Awesome Todo App"

- Re-build image again and see the differences

```bash
docker build -t <login-id>/getting-started .
```

We can notice that the build was much faster, and there're few steps are using previously cached layers.

## Part 9: What next