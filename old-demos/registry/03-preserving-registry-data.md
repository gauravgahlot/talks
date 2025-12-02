# Preserving the Registry Data in a Docker Volume

In this lab session, we will learn about preserving the registry data in a Docker Volume. This ensures that we don't loose our data even after the server (registry container) is stopped and removed. 

## Stop the Registry server

- Stop and remove the registry server using the following commands
```
docker container stop registry
docker container rm registry
```
- Restart the registry server using the following command
```
docker run -d -p 5000:5000 --name registry registry:2.7
```
- Go to **http://REGISTRY-HOST-IP:5000/v2/_catalog/** and observe that we have lost the repositories

## Preserving Registry Data in a Docker Volume

- Stop and remove the registry server with above commands
- Restart the registry server and add a volume with the following command
```
docker run -d -p 5000:5000 -v registry-data:/var/lib/registry --name registry registry:2.7
```
- Note that **registry-data** is the named volume that will be created for us. We can name it anything we want. However, it must be mounted at **/var/lib/registry**.
- Push the image again with the above used commands

## Testing 

- Stop and remove the registry server with above commands
- Execute the command **docker volume ls** to check if volume is not deleted
- Restart the registry server and add the same volume with the following command
```
docker run -d -p 5000:5000 -v registry-data:/var/lib/registry --name registry registry:2.7
```
- Go to **http://REGISTRY-HOST-IP:5000/v2/_catalog/** and observe that our repositories are still there.
<br>

[Next > Securing Registry with Basic Auth](https://github.com/QuickDevNotes/Docker-Labs/blob/master/registry/04-securing-registry-with-basic-auth.md)
[Previous > Pulling an Image](https://github.com/QuickDevNotes/Docker-Labs/blob/master/registry/02-pull-an-image-from-registry.md)
