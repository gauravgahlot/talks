# Adding a User Interface to our Registry 

In this lab session we will set a clean user interface for our Registry. For this purpose I will be using the **konradkleine/docker-registry-frontend** image available at [DockerHub](https://hub.docker.com/r/konradkleine/docker-registry-frontend). There are different options that we can choose from, however, I find this one pretty clean.

- We can add the frontend to registry to our registry by running a container that hosts the UI, with the following command
```
docker run \
  -d \
  -e ENV_DOCKER_REGISTRY_HOST=YOUR-REGISTRY-HOST \
  -e ENV_DOCKER_REGISTRY_PORT=5000 \
  -p 8080:80 \
  konradkleine/docker-registry-frontend:v2
```
Note that `ENV_DOCKER_REGISTRY_PORT` might change if you are registry server is listening at a different port.
- Open a browser and go to **http://REGISTRY-HOST-IP:8080/**
- Provide login username and password for the registry that we had configured for [basic authentication](https://github.com/QuickDevNotes/Docker-Labs/blob/master/registry/04-securing-registry-with-basic-auth.md)
<br>

[Previous > Securing our Registry with Basic Auth](https://github.com/QuickDevNotes/Docker-Labs/blob/master/registry/04-securing-registry-with-basic-auth.md)
