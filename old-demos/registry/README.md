# Docker Registry
<br>

## Prerequisite

### Infra

- Good to have have two machines. 
- We can either use two VMs or two instances from a cloud provider (AWS), that can communicate with each other. This set up would be more suitable to run through the lab. 

### Docker 

- Log into the two instances and ensure that Docker is installed. 
- Execute **docker info** command to ensure that the user has required permissions. 
- If there is an error, we can switch to **sudo** user and continue.
- A better approach is to add current user to **docker** group using the following command:
```
sudo usermod -aG docker $USER
```
Now, logout and then login. 
<br>

Congratulations! We have successfully setup our environment for the lab sessions.
<br>


## Labs

- [Running the first Registry Server](https://github.com/QuickDevNotes/Docker-Labs/blob/master/registry/01-running-first-registry-server.md)
- [Pulling an Image from Registry Server](https://github.com/QuickDevNotes/Docker-Labs/blob/master/registry/02-pull-an-image-from-registry.md)
- [Preserving registry data in a named Docker Volume](https://github.com/QuickDevNotes/Docker-Labs/blob/master/registry/03-preserving-registry-data.md)
- [Securing the Registry with Basic-Auth (htpasswd)](https://github.com/QuickDevNotes/Docker-Labs/blob/master/registry/04-securing-registry-with-basic-auth.md)
- [Adding a User Interface to Registry](https://github.com/QuickDevNotes/Docker-Labs/blob/master/registry/05-add-registry-UI.md)
- [Docker Registry as Pull Through Cache](https://github.com/QuickDevNotes/Docker-Labs/tree/master/registry/registry-pull-through-cache)
