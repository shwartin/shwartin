# Docker quick start

запустить контейнер hello-dock на порту 8080:
```
docker container run --publish 8080:80 fhsinchy/hello-dock
```

Запустить (создать и запустить) контейнер в бекграунде:
```
docker container run --detach --publish 8080:80 fhsinchy/hello-dock
```

Посмотреть контейнеры:
```
docker container ls
```

Посмотреть все контейнеры, включая остановленные:
```
docker container ls --all
```

Задать имя контейнеру
```
docker container run --detach --publish 8888:80 --name hello-dock-container fhsinchy/hello-dock
```

Переименовать контейнер
```
docker container rename gifted_sammet hello-dock-container-2
```

Остановить контейнер, можно задавать имя или айди (`SIGTERM`)
```
docker container stop hello-dock-container
```

Убить контейнер (`SIGKILL`)
```
docker container kill hello-dock-container-2
```

Запустить контейнер
```
docker container start hello-dock-container
```

Создать контейнер
```
docker container create --publish 8080:80 fhsinchy/hello-dock
```

Рестартовать контейнер
```
docker container restart hello-dock-container-2
```

Удалить контейнер
```
docker container rm 6cf52771dde1
```

Создать и запустить контейнер в бекграунде на порту 8888, задать имя и удалить после остановки
```
docker container run --rm --detach --publish 8888:80 --name hello-dock-volatile fhsinchy/hello-dock
```

Остановить контейнер
```
docker container stop hello-dock-volatile
```

Запустить контейнер в интерактивном режиме
```
docker container run --rm -it ubuntu
```

## Запуск команд
Выполнить команду `uname -a` внутри контейнера
```
docker run alpine uname -a
```

Выполнить баш команду в контейнере
```
docker container run --rm busybox sh -c "echo -n my-secret | base64"
```

bind mount in directory /zone
```
docker container run --rm -v $(pwd):/zone fhsinchy/rmbyext pdf
```


## How to create docker image
Dockerfile demo
```dockerfile
FROM ubuntu:latest

EXPOSE 80

RUN apt-get update && \
    apt-get install nginx -y && \
    apt-get clean && rm -rf /var/lib/apt/lists/*

CMD ["nginx", "-g", "daemon off;"]
```

To buid image
```
docker image build .
```

Run image
```
docker container run --rm --detach --name custom-nginx-packaged --publish 8080:80 3199372aa3fc
```

Add tag to image
```
docker image build --tag custom-nginx:packaged .
```

## List and remove docker images
```
docker image ls
```

Delete image
```
docker image rm custom-nginx:packaged
```

Cleanup all un-tagged images
```
docker image prune --force
```

## Docker image layers
Visualize image layers. The upper most layer is the latest.
```
docker image history custom-nginx:packaged
```

## Build nginx from source
-   Get a good base image for building the application, like [ubuntu](https://hub.docker.com/_/ubuntu).
-   Install necessary build dependencies on the base image.
-   Copy the `nginx-1.19.2.tar.gz` file inside the image.
-   Extract the contents of the archive and get rid of it.
-   Configure the build, compile and install the program using the `make` tool.
-   Get rid of the extracted source code.
-   Run `nginx` executable.

```
FROM ubuntu:latest

RUN apt-get update && \
    apt-get install build-essential\ 
                    libpcre3 \
                    libpcre3-dev \
                    zlib1g \
                    zlib1g-dev \
                    libssl1.1 \
                    libssl-dev \
                    -y && \
    apt-get clean && rm -rf /var/lib/apt/lists/*

COPY nginx-1.19.2.tar.gz .

RUN tar -xvf nginx-1.19.2.tar.gz && rm nginx-1.19.2.tar.gz

RUN cd nginx-1.19.2 && \
    ./configure \
        --sbin-path=/usr/bin/nginx \
        --conf-path=/etc/nginx/nginx.conf \
        --error-log-path=/var/log/nginx/error.log \
        --http-log-path=/var/log/nginx/access.log \
        --with-pcre \
        --pid-path=/var/run/nginx.pid \
        --with-http_ssl_module && \
    make && make install

RUN rm -rf /nginx-1.19.2

CMD ["nginx", "-g", "daemon off;"]
```

Build image
```
docker image build --tag custom-nginx:built .
```

Add arguments
```dockerfile
FROM ubuntu:latest

RUN apt-get update && \
    apt-get install build-essential\ 
                    libpcre3 \
                    libpcre3-dev \
                    zlib1g \
                    zlib1g-dev \
                    libssl1.1 \
                    libssl-dev \
                    -y && \
    apt-get clean && rm -rf /var/lib/apt/lists/*

ARG FILENAME="nginx-1.19.2"
ARG EXTENSION="tar.gz"

ADD https://nginx.org/download/${FILENAME}.${EXTENSION} .

RUN tar -xvf ${FILENAME}.${EXTENSION} && rm ${FILENAME}.${EXTENSION}

RUN cd ${FILENAME} && \
    ./configure \
        --sbin-path=/usr/bin/nginx \
        --conf-path=/etc/nginx/nginx.conf \
        --error-log-path=/var/log/nginx/error.log \
        --http-log-path=/var/log/nginx/access.log \
        --with-pcre \
        --pid-path=/var/run/nginx.pid \
        --with-http_ssl_module && \
    make && make install

RUN rm -rf /${FILENAME}}

CMD ["nginx", "-g", "daemon off;"]
```

Run container
```
docker container run --rm --detach --name custom-nginx-built --publish 8080:80 custom-nginx:built
```

## Otimize docker image
Remove packeges after nginx build. It need in the same layer.
```dockerfile
FROM ubuntu:latest

EXPOSE 80

ARG FILENAME="nginx-1.19.2"
ARG EXTENSION="tar.gz"

ADD https://nginx.org/download/${FILENAME}.${EXTENSION} .

RUN apt-get update && \
    apt-get install build-essential \ 
                    libpcre3 \
                    libpcre3-dev \
                    zlib1g \
                    zlib1g-dev \
                    libssl1.1 \
                    libssl-dev \
                    -y && \
    tar -xvf ${FILENAME}.${EXTENSION} && rm ${FILENAME}.${EXTENSION} && \
    cd ${FILENAME} && \
    ./configure \
        --sbin-path=/usr/bin/nginx \
        --conf-path=/etc/nginx/nginx.conf \
        --error-log-path=/var/log/nginx/error.log \
        --http-log-path=/var/log/nginx/access.log \
        --with-pcre \
        --pid-path=/var/run/nginx.pid \
        --with-http_ssl_module && \
    make && make install && \
    cd / && rm -rfv /${FILENAME} && \
    apt-get remove build-essential \ 
                    libpcre3-dev \
                    zlib1g-dev \
                    libssl-dev \
                    -y && \
    apt-get autoremove -y && \
    apt-get clean && rm -rf /var/lib/apt/lists/*

CMD ["nginx", "-g", "daemon off;"]
```

Using alpine linux
```dockerfile
FROM alpine:latest

EXPOSE 80

ARG FILENAME="nginx-1.19.2"
ARG EXTENSION="tar.gz"

ADD https://nginx.org/download/${FILENAME}.${EXTENSION} .

RUN apk add --no-cache pcre zlib && \
    apk add --no-cache \
            --virtual .build-deps \
            build-base \ 
            pcre-dev \
            zlib-dev \
            openssl-dev && \
    tar -xvf ${FILENAME}.${EXTENSION} && rm ${FILENAME}.${EXTENSION} && \
    cd ${FILENAME} && \
    ./configure \
        --sbin-path=/usr/bin/nginx \
        --conf-path=/etc/nginx/nginx.conf \
        --error-log-path=/var/log/nginx/error.log \
        --http-log-path=/var/log/nginx/access.log \
        --with-pcre \
        --pid-path=/var/run/nginx.pid \
        --with-http_ssl_module && \
    make && make install && \
    cd / && rm -rfv /${FILENAME} && \
    apk del .build-deps

CMD ["nginx", "-g", "daemon off;"]
```

## Share docker image online
```
docker login

docker image build --tag fhsinchy/custom-nginx:latest --file Dockerfile.built .

docker image push fhsinchy/custom-nginx:latest

```

## How ot work with bind mounts
```
docker container run \
    --rm \
    --detach \
    --publish 3000:3000 \
    --name hello-dock-dev \
    --volume $(pwd):/home/node/app \
    --volume /home/node/app/node_modules \
    hello-dock:dev
```

## Multistaged builds
Remove node after build
```dockerfile
FROM node:lts-alpine as builder

WORKDIR /app

COPY ./package.json ./
RUN npm install

COPY . .
RUN npm run build

FROM nginx:stable-alpine

EXPOSE 80

COPY --from=builder /app/dist /usr/share/nginx/html
```

## Network manipulation
Inspect  `notes-api-db-server` container IP
```
docker container inspect --format='{{range .NetworkSettings.Networks}} {{.IPAddress}} {{end}}' notes-api-db-server
```

By default, Docker has five networking drivers. They are as follows:

-   `bridge` - The default networking driver in Docker. This can be used when multiple containers are running in standard mode and need to communicate with each other.
-   `host` - Removes the network isolation completely. Any container running under a `host` network is basically attached to the network of the host system.
-   `none` - This driver disables networking for containers altogether. I haven't found any use-case for this yet.
-   `overlay` - This is used for connecting multiple Docker daemons across computers and is out of the scope of this book.
-   `macvlan` - Allows assignment of MAC addresses to containers, making them function like physical devices in a network.

-   **User-defined bridges provide automatic DNS resolution between containers:** This means containers attached to the same network can communicate with each others using the container name. So if you have two containers named `notes-api` and `notes-db` the API container will be able to connect to the database container using the `notes-db` name.
-   **User-defined bridges provide better isolation:** All containers are attached to the default bridge network by default which can cause conflicts among them. Attaching containers to a user-defined bridge can ensure better isolation.
-   **Containers can be attached and detached from user-defined networks on the fly:** During a container’s lifetime, you can connect or disconnect it from user-defined networks on the fly. To remove a container from the default bridge network, you need to stop the container and recreate it with different network options.

## Create network 
```
docker network create skynet
```

## Connect container to network
connect the `hello-dock` container to the `skynet` network
```
docker network connect skynet hello-dock

docker network inspect --format='{{range .Containers}} {{.Name}} {{end}}' skynet

#  hello-dock

docker network inspect --format='{{range .Containers}} {{.Name}} {{end}}' bridge

#  hello-dock
```


another `hello-dock` container attached to the same network
```
docker container run --network skynet --rm --name alpine-box -it alpine sh

# lands you into alpine linux shell

/ # ping hello-dock

# PING hello-dock (172.18.0.2): 56 data bytes
# 64 bytes from 172.18.0.2: seq=0 ttl=64 time=0.191 ms
# 64 bytes from 172.18.0.2: seq=1 ttl=64 time=0.103 ms
# 64 bytes from 172.18.0.2: seq=2 ttl=64 time=0.139 ms
# 64 bytes from 172.18.0.2: seq=3 ttl=64 time=0.142 ms
# 64 bytes from 172.18.0.2: seq=4 ttl=64 time=0.146 ms
# 64 bytes from 172.18.0.2: seq=5 ttl=64 time=0.095 ms
# 64 bytes from 172.18.0.2: seq=6 ttl=64 time=0.181 ms
# 64 bytes from 172.18.0.2: seq=7 ttl=64 time=0.138 ms
# 64 bytes from 172.18.0.2: seq=8 ttl=64 time=0.158 ms
# 64 bytes from 172.18.0.2: seq=9 ttl=64 time=0.137 ms
# 64 bytes from 172.18.0.2: seq=10 ttl=64 time=0.145 ms
# 64 bytes from 172.18.0.2: seq=11 ttl=64 time=0.138 ms
# 64 bytes from 172.18.0.2: seq=12 ttl=64 time=0.085 ms

--- hello-dock ping statistics ---
13 packets transmitted, 13 packets received, 0% packet loss
round-trip min/avg/max = 0.085/0.138/0.191 ms
```

for the automatic DNS resolution to work you must assign custom names to the containers

## Disconnect container from network
To detach the `hello-dock` container from the `skynet` network
```
docker network disconnect skynet hello-dock
```

Remove network
```
docker network rm skynet
```

## Multi-Container docker application
Run database server, but data will be inside 
```
docker container run \
    --detach \
    --name=notes-db \
    --env POSTGRES_DB=notesdb \
    --env POSTGRES_PASSWORD=secret \
    --network=notes-api-network \
    postgres:12

# a7b287d34d96c8e81a63949c57b83d7c1d71b5660c87f5172f074bd1606196dc

docker container ls

# CONTAINER ID   IMAGE         COMMAND                  CREATED              STATUS              PORTS      NAMES
# a7b287d34d96   postgres:12   "docker-entrypoint.s…"   About a minute ago   Up About a minute   5432/tcp   notes-db
```


Create volume
```
docker volume create notes-db-data

# notes-db-data

docker volume ls

# DRIVER    VOLUME NAME
# local     notes-db-data
```

Run database server with mounted volume
```
docker container run \
    --detach \
    --volume notes-db-data:/var/lib/postgresql/data \
    --name=notes-db \
    --env POSTGRES_DB=notesdb \
    --env POSTGRES_PASSWORD=secret \
    --network=notes-api-network \
    postgres:12

# 37755e86d62794ed3e67c19d0cd1eba431e26ab56099b92a3456908c1d346791
```

Check mounting
```
docker container inspect --format='{{range .Mounts}} {{ .Name }} {{end}}' notes-db

#  notes-db-data
```

Create network
```
docker network create notes-api-network
```

attach the `notes-db` container to this network
```
docker network connect notes-api-network notes-db
```

Create dockerfile for javascript
```
# stage one
FROM node:lts-alpine as builder

# install dependencies for node-gyp
RUN apk add --no-cache python make g++

WORKDIR /app

COPY ./package.json .
RUN npm install --only=prod

# stage two
FROM node:lts-alpine

EXPOSE 3000
ENV NODE_ENV=production

USER node
RUN mkdir -p /home/node/app
WORKDIR /home/node/app

COPY . .
COPY --from=builder /app/node_modules  /home/node/app/node_modules

CMD [ "node", "bin/www" ]
```

Build image
```
docker image build --tag notes-api .
```

Check database container is runing and attached to the network
```
docker container inspect notes-db
```

Final docker run
```
docker container run \
    --detach \
    --name=notes-api \
    --env DB_HOST=notes-db \
    --env DB_DATABASE=notesdb \
    --env DB_PASSWORD=secret \
    --publish=3000:3000 \
    --network=notes-api-network \
    notes-api
    
# f9ece420872de99a060b954e3c236cbb1e23d468feffa7fed1e06985d99fb919
```

Check
```
docker container ls
```

## Access logs
```
docker container logs notes-db
```

## Execute commands in a runnig container
To execute `npm run db:migrate` inside the `notes-api` container

```
docker container exec notes-api npm run db:migrate

# > notes-api@ db:migrate /home/node/app
# > knex migrate:latest
#
# Using environment: production
# Batch 1 run: 1 migrations
```

run an interactive command inside
```
docker container exec -it notes-api sh

# / # uname -a
# Linux b5b1367d6b31 5.10.9-201.fc33.x86_64 #1 SMP Wed Jan 20 16:56:23 UTC 2021 x86_64 Linux
```


## Docker-compose for multi-container application
Although Compose works in all environments, it's more focused on development and testing. Using Compose on a production environment is not recommended at all.

```yaml
version: "3.8"

services: 
    db:
        image: postgres:12
        container_name: notes-db-dev
        volumes: 
            - notes-db-dev-data:/var/lib/postgresql/data
        environment:
            POSTGRES_DB: notesdb
            POSTGRES_PASSWORD: secret
    api:
        build:
            context: ./api
            dockerfile: Dockerfile.dev
        image: notes-api:dev
        container_name: notes-api-dev
        environment: 
            DB_HOST: db ## same as the database service name
            DB_DATABASE: notesdb
            DB_PASSWORD: secret
        volumes: 
            - /home/node/app/node_modules
            - ./api:/home/node/app
        ports: 
            - 3000:3000

volumes:
    notes-db-dev-data:
        name: notes-db-dev-data
```


Run. make sure you've opened your terminal in the same directory where the `docker-compose.yaml`
```
docker-compose --file docker-compose.yaml up --detach
```

command for listing containers defined in the YAML only.
```
docker-compose ps
```

## Execute commands insode a runnig service in docker compose

```
docker-compose exec api npm run db:migrate
```

## Access logs in docker compose
```
docker-compose logs api
```

## Stop services
```
docker-compose down --volumes
```


## Full-stack application using docker compose
<img src="https://www.freecodecamp.org/news/content/images/2021/01/fullstack-application-design.svg">


```yaml
version: "3.8"

services: 
    db:
        image: postgres:12
        container_name: notes-db-dev
        volumes: 
            - db-data:/var/lib/postgresql/data
        environment:
            POSTGRES_DB: notesdb
            POSTGRES_PASSWORD: secret
        networks:
            - backend
    api:
        build: 
            context: ./api
            dockerfile: Dockerfile.dev
        image: notes-api:dev
        container_name: notes-api-dev
        volumes: 
            - /home/node/app/node_modules
            - ./api:/home/node/app
        environment: 
            DB_HOST: db ## same as the database service name
            DB_PORT: 5432
            DB_USER: postgres
            DB_DATABASE: notesdb
            DB_PASSWORD: secret
        networks:
            - backend
    client:
        build:
            context: ./client
            dockerfile: Dockerfile.dev
        image: notes-client:dev
        container_name: notes-client-dev
        volumes: 
            - /home/node/app/node_modules
            - ./client:/home/node/app
        networks:
            - frontend
    nginx:
        build:
            context: ./nginx
            dockerfile: Dockerfile.dev
        image: notes-router:dev
        container_name: notes-router-dev
        restart: unless-stopped
        ports: 
            - 8080:80
        networks:
            - backend
            - frontend

volumes:
    db-data:
        name: notes-db-dev-data

networks: 
    frontend:
        name: fullstack-notes-application-network-frontend
        driver: bridge
    backend:
        name: fullstack-notes-application-network-backend
        driver: bridge
```

Start all
```
docker-compose --file docker-compose.yaml up --detach
```

## References

About DCA
https://medium.com/bb-tutorials-and-thoughts/all-you-need-to-know-about-docker-certified-associate-dca-exam-21dd2ccadbc0

250 questions DCA
https://medium.com/bb-tutorials-and-thoughts/250-practice-questions-for-the-dca-exam-84f3b9e8f5ce

Play with docker
https://training.play-with-docker.com/

Курс подготовки к DCA
https://www.udemy.com/course/docker-certified-associate/

https://www.freecodecamp.org/news/the-docker-handbook/
