# Pulling an Image from Registry Server

In this lab session, we will learn about how to add our _insecure_ registry to different Docker hosts and pull an image.

## The First Attempt

- Go to second Docker host
- Execute the following command to pull the image we just pushed to our registry
```
docker pull <REGISTRY-HOST-IP>:5000/my-busybox

Using default tag: latest
Error response from daemon: Get https://<REGISTRY-HOST-IP>:5000/v2/: http: server gave HTTP response to HTTPS client
```
- What's wrong? 
- Execute **docker info** command to get Docker daemon details and notice **Insecure Registries**.
  
## Add the Registry Server

- Execute the following command to create the config file for Docker **daemon**
```
nano /etc/docker/daemon.json
```
- Add the following JSON
```
{
  "insecure-registries" : ["REGISTRY-HOST-IP:5000"]
}
```
- Restart the Docker daemon by executing the command **sudo service docker restart**
- Retry to pull the image
```
docker pull <REGISTRY-HOST-IP>:5000/my-busybox
```
<br>

[Next > Preserving registry data](https://github.com/QuickDevNotes/Docker-Labs/blob/master/registry/03-preserving-registry-data.md)
[Previous > Running a Registry Server](https://github.com/QuickDevNotes/Docker-Labs/blob/master/registry/01-running-first-registry-server.md)
