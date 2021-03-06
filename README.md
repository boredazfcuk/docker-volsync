# docker-volsync

A container that synchronises volumes within a Swarm cluster asynchronously. In fact, there are some scenarios 
where you want keep synchronised some volume across the Swarm cluster. For example,
if you want ensure a simple high availability.

The files are usually synchronized within 5 seconds.

## Components
It requires Lsync and Csync2 and designed to be deployed inside a Docker Stack.

## Image
You can download the image using `docker pull boredazfcuk/volsync`

## Prerequisities
The container requires a docker Swarm cluster. It cannot sync volumes for separate containers. Also, it must be deployed in the 
same stack as the service you wish to synchronise.

## Usage
I suggest to create a dedicated network beacause the ingress network doesn't do the service discovery.
The service has to be created mounting the Docker socket (`/var/run/docker.sock`) and the volumes to synchronize.
Then, you should pass a random key (only A-Z a-z 0-9 digit) as env var CSYNC2_KEY.
In the deploy of this image you should define the service to synchronize with the label `async.service`.
In the deploy of the image to sync you should define the label `async.volumes` with the name of the volume to sync.

### Example
An example of a stack that use this image.
In this case the service to synchronize is `jenkins`, so `async.service: 'jenkins'`
in `services->async->deploy->labels`.
The volumes to sync are `wp_content_async jenkins_home`, so `async.volumes: 'wp_content_async,jenkins_home'`
in `services->jenkins->deploy->labels`
Remember to mount the volumes on both services.

```
version: '3.3'
services:
  jenkins:
    image: jenkins/jenkins
    deploy:
      replicas: 1
      labels: 
        async.volumes: 'wp_content_async,jenkins_home'
        traefik.frontend.rule: 'Host:fabrizio.no-ip.dynu.net'
    ports:
      - "8080:8080"
    volumes:
      - jenkins_home:/var/jenkins_home
      - wp_content_async:/var/www/html
    restart: always
    networks: 
      - network-async
  async:
    image: fabrizio2210/docker_volume_synchronizer
    deploy:
      mode: global
      labels:
        async.service: 'jenkins'
    volumes:
      - jenkins_home:/opt/data
      - wp_content_async:/opt/data2
      - /var/run/docker.sock:/var/run/docker.sock
    environment:
      CSYNC2_KEY: dasbdsfsdfsdfn12dsfsdfsdfsdbc9089nfdg24342hjgfdsa
     
volumes:
  jenkins_home:
    driver: 'local'
    driver_opts: 
      type: 'none'
      o: 'bind'
      device: "/var/mount1/"
  wp_content_async:
    driver: 'local'
    driver_opts: 
      type: 'none'
      o: 'bind'
      device: "/var/mount2/"
networks:
  network-async:
    external: true
```
These are the commands to use this stack.
```
docker swarm init
docker network create --driver overlay network-sync
docker stack deploy -c tests/stack.yml test-stack
```


### Test

To test this project you should install Vagrant.
After that you can use the following commands.

```
vagrant-tools/setup.sh
tests/sync_with_stack.sh
```
