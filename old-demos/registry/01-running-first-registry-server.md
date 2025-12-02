# Running the first Registry Server

In this lab session we will run our very first Registry Server based on the **registry:2.7** image available on [Docker Hub](https://hub.docker.com/_/registry).

## Spin-up the Registry Server

- Log into the first Docker host.
- Make a note of IP address of this host. We can get the IP, using **ifconfig** command.
- Execute following command
```
docker run -d -p 5000:5000 --name registry registry:2.7
```

## Interacting with the Registry Server

- Open a browser and go to **http://REGISTRY-HOST-IP:5000/v2/_catalog/**
- This returns the list of **Repositories** as a JSON response, and it should look like
```
// http://REGISTRY-HOST-IP:5000/v2/_catalog
{
  "repositories": [
  ]
}
```
<br>


## Push a Docker Image to Registry 

In this lab session we will build a Docker image and then push it to the Registry.

### Build a Docker Image

- Create a Dockerfile using the **nano Dockerfile**
- Add the following commands in the Dockerfile
```
FROM busybox
LABEL Author="<YOUR-NAME>"
LABEL Event="DockerPune Meetup"
```
- Execute the following command to build the image 
```
docker build -t <REGISTRY-HOST-IP>:5000/my-busybox .
```
- Execute **docker images** to verify the image 

### Push the image

- Execute the following command to tag the image
```
docker tag <REGISTRY-HOST-IP>:5000/my-busybox <REGISTRY-HOST-IP>:5000/my-busybox:v1
docker tag <REGISTRY-HOST-IP>:5000/my-busybox <REGISTRY-HOST-IP>:5000/my-busybox:custom
```
- Execute the command to push the image
```
docker push <REGISTRY-HOST-IP>:5000/my-busybox
```

### Verify the push

- Open a browser and go to **http://REGISTRY-HOST-IP:5000/v2/_catalog/**
- This returns the list of **Repositories** as a JSON response, and it should look like
```
// http://FIRST-HOST-IP:5000/v2/_catalog
{
  "repositories": [
    "my-busybox"
  ]
}
```
- Go to **http://REGISTRY-HOST-IP:5000/v2/my-busybox/tags/list/** to get list of image tags
```
// http://FIRST-HOST-IP:5000/v2/my-busybox/tags/list
{
  "name": "my-busybox",
  "tags": [
    "custom"
    "latest",
    "v1"
  ]
}
```
<br>

[Next > Pulling an Image](https://github.com/QuickDevNotes/Docker-Labs/blob/master/registry/02-pull-an-image-from-registry.md)
