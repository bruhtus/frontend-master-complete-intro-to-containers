# Instructions

Use this command to build the docker image from Dockerfile:
```sh
docker build -t my-node-app .
```

Use this command to run the docker container:
```sh
docker run --rm --name node-app --init -p 3000:3000 my-node-app:latest
```

Use docker compose instead of docker run:
```sh
docker-compose up
```
or
```sh
docker-compose up -d
```
