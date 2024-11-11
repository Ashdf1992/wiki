# Docker Cheatsheet

<br>

## List Docker Images:

``` Bash
docker images
```

<br>

## Update Images
> First go to the location of where 'docker-compose.yml' is located for instance /docker/nginx
``` Bash
docker compose pull
```

<br>

## Stop Image
``` Bash
docker stop nginx
```

<br>

## Start the New Image
``` Bash
docker docker-compose.yaml up -d
```

<br>

## List images
``` Bash
docker image list
```

<br>

## List Containers
``` Bash
docker container list
```

<br>

## Remove Image
``` Bash
docker rmi aaaabbbbcccc
```

<br>

## Remove Container
``` Bash
docker container remove aaaabbbbcccc
```

<br>

## Prune Images
``` Bash
docker image prune
```

