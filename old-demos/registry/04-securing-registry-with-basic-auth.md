# Securing our Registry with Basic Auth

In this lab session we will learn about how we can secure our registry using basic authentication.

## Build new Registry Image

- Stop and remove the registry server with above commands
- Creata a new Dockerfile with **nano Dockerfile** command and the following commands to it:
```
FROM registry:2.7

# store the username and password in /auth/htpasswd file
RUN mkdir /auth && htpasswd -bnB admin admin > /auth/htpasswd

# set the environment variables for registry
ENV REGISTRY_AUTH htpasswd
ENV REGISTRY_AUTH_HTPASSWD_REALM basic-realm
ENV REGISTRY_AUTH_HTPASSWD_PATH /auth/htpasswd

# expose port for registry
EXPOSE 5000
```
- Build an image using the following command
```
docker build -t secure-registry .
```
- Start the registry server with new image, using the following command
```
docker run -d -p 5000:5000 -v registry-data:/var/lib/registry --name registry secure-registry
```
- Go to **http://REGISTRY-HOST-IP:5000/v2/_catalog/** and observe that now we are prompted for username and password.

## Push an Image

- Let's try to push the image we had built earlier, and this time we will fail
```
docker push localhost:5000/my-busybox

The push refers to repository [localhost:5000/my-busybox]
adab5d09ba79: Preparing 
no basic auth credentials
```
- Login to the registry using the following command, and provide username and password as prompted
```
docker login localhost:5000
```
- Retry to push the image now

## Pull an Image

- Let's try to pull an image on second host, and we will fail here as well
- Login to the registry using the following command, and provide username and password as prompted
```
docker login localhost:5000
```
- Retry to pull the image now
<br>

[Next > Adding a User Interface](https://github.com/QuickDevNotes/Docker-Labs/blob/master/registry/05-add-registry-UI.md)
[Previous > Preserving registry data](https://github.com/QuickDevNotes/Docker-Labs/blob/master/registry/03-preserving-registry-data.md)
