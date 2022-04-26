# Docker Swarm Mode setup

After initial [Ansible playbook](../README.md) is being rolled out, you can setup the
swarm. I prefer running all the things from my local machine instead of SSHing
to the manager, then to the worker etc. etc. So, assuming you've got `docker`
command at your disposal:

```bash
docker context create --docker host=ssh://root@swarm-lb-1.chbk.co --description "Docker Swarm Manager 1" mng-1
docker context create --docker host=ssh://root@swarm-lb-2.chbk.co --description "Docker Swarm Manager 2" mng-2
docker context create --docker host=ssh://root@swarm-lb-3.chbk.co --description "Docker Swarm Manager 3" mng-3
docker context use mng-1
docker info
```

The same can be done for the worker:

```bash
docker context create --docker host=ssh://root@swarm-wrk-1.chbk.co --description "Docker Swarm Worker 1" wrk-1
docker context create --docker host=ssh://root@swarm-wrk-2.chbk.co --description "Docker Swarm Worker 2" wrk-2
docker context create --docker host=ssh://root@swarm-wrk-3.chbk.co --description "Docker Swarm Worker 3" wrk-3
docker context create --docker host=ssh://root@swarm-wrk-4.chbk.co --description "Docker Swarm Worker 4" wrk-4
docker context use wrk-1
docker info
```

Switching between the two is now as easy as `docker context use $NAME`. You can
list all of the available contexts via `docker context ls`.

All of the commands are running via SSH tunnel 🔒


## Setting up the cluster

As can be observed in the previous section, this cluster consists of 3 manager
and 4 worker nodes. They were provisioned on DigitalOcean, but the entire setup
should work on any given VPS provider. DO provides feature called VPC that
allows for all the nodes from the same region to communicate internally (LAN).
For that reason I'm passing the `--advertise-addr eth1` so that internal network
is used for communication of the Swarm cluster.

On one of the manager nodes (I'm assuming `mng-1` here):

```bash
docker swarm init --advertise-addr eth1
```

This command will spit out `docker swarm join[...]` command with the token that
can be then/later used on the workers to join the cluster.

On the rest nodes:

```bash
docker swarm join --token $TOKEN 10.133.0.4:2377
```

Please note: the `$TOKEN` needs to be replaced with whatever the `swarm init`
spit out previously.

OK, as all of the reminder nodes were added as workers, let's quickly elevate
two of them to have 3 managers in the cluster:

```
docker node promote mng-2 mng-3
```

Cool, the `docker node ls` should now spit out something like this:

```bash
ID                            HOSTNAME      STATUS    AVAILABILITY   MANAGER STATUS   ENGINE VERSION
qm44ap8ey7ynciylm46lcwe7l *   swarm-lb-1    Ready     Active         Leader           20.10.14
9y6ocfcxzj9f0clwou9xnuabz     swarm-lb-2    Ready     Active         Reachable        20.10.14
qypv72xf0hsrsforph4wt1wmr     swarm-lb-3    Ready     Active         Reachable        20.10.14
u5yxw9stdi8ar56f4ljv8h1tx     swarm-wrk-1   Ready     Active                          20.10.14
p75dcvsckrd5dhe3pqly32m6f     swarm-wrk-2   Ready     Active                          20.10.14
kuxn6apc6mrnfimhldlph6vyy     swarm-wrk-3   Ready     Active                          20.10.14
owpgayiqyn8zhyaf0bjvxk7tw     swarm-wrk-4   Ready     Active                          20.10.14
```

Cluster is assembled, job well done 🍺


## Preparing for the deployments

I'm going to deploy multiple instances of [Ghost](https://ghost.org) and I'm gonna use
[Traefik](https://traefik.io) as a LB and MySQL for the database. While setting up the DNS and
TLS certificates is out of the scope of this document, but the things worth
noting are: I'm using wildcard domain `*.swarm.chbk.co` and a wildcard
certificate for it from Let's Encrypt.

### Secrets

Let's start with creating secrets — two for TLS certificate and the key and
third one with the password for MySQL:

```bash
docker secret create fullchain.pem fullchain.pem
docker secret create privkey.pem privkey.pem
openssl rand -base64 20 | docker secret create db_root_password -
```

Cool, there should be three secrets now set, let's check via `docker secret ls`:

```bash
ID                          NAME               DRIVER    CREATED        UPDATED
ia1h2bvrl40waq606sbvzklrb   db_root_password             22 hours ago   22 hours ago
svsq0u3ybparldw9i30bog2rr   fullchain.pem                26 hours ago   26 hours ago
xf3ev3jz3nh32j1j93v22bbtu   privkey.pem                  26 hours ago   26 hours ago
```

### Configuration

`docker config` works very similarly to `docker secret`, the only difference
being the encryption at rest part (i.e. configs are not encrypted). What's cool
is that whatever is stored in the `docker secret` or `config` is being available
for the entire cluster. TLS key and cert + configuration for Traefik will then
be shared between three manager nodes on which Traefik LB is going to run. Let's
add the config then:

```bash
docker config create certificates.yaml certificates.yaml
```

This file is being part of this repo, you can view it here
[certificates.yaml](certificates.yaml).

### Network

If you don't really care, just smash everything into `ingress` and call it a
day. I wanted to separate things a bit cleaner so I created two networks:
`public` for handling external traffic via Traefik and `db` to handle connection
between Ghost instance and MySQL database:

```bash
docker network create -d overlay public
docker network create -d overlay db
```

### Labels

One of the nice features of Swarm is that it allows for setting up your own 
`labels` that you can later on use to constraint where which containers can
operate/run on. Good example for this is when you need to use a volume to
persist data of, let's say, MySQL. In order for this to work, one have to always
start and or restart this database on the same host where the volume is
available. One way to achieve this is to set a label on the host where the
volume will get created on the frist run of MySQL and constraint it to always
run there. The constraint is set in the [db.yml](db.yml) stack file, here's the
label setup:

```bash
docker node update --label-add db-data=true swarm-wrk-1
```

I chose worker node 1 in this case to serve as the only node where DB can run.


## Deployments

Now that everything's ready, it's time to simply start all the things.

### Traefik

```bash
docker stack deploy -c swarm/traefik.yml traefik
```

Traefik is a global service, which means that it will run on all of the
available nodes, however, I am constraining it to only manager nodes. If any of
the workers ever gets promoted to a manager role or entirely new manager gets
introduced to the cluster, Traefik spin there up automatically 🪄

If everything went as planned, Traefik dashboard should be now available here:
https://traefik.swarm.chbk.co. It should also have valid certificates and
automatically redirect all http requests to https.

### MySQL

```bash
docker stack deploy -c swarm/db.yml db
```

This one gets constrained to only worker nodes **and** only the ones that have
`db-data` label. Neat.

### Ghost

Sadly, Ghost Docker setup doesn't support `docker secret` so I reverted to using
environment variables when spinning things up:

```bash
env GHOST=ghost DB_PASSWORD=password docker stack deploy -c swarm/ghost.yml ghost
```

Cool thing to note: the above with different `GHOST=ghost` and stack name (end
of the command) can be spun up many times and it will just work, getting domain
with the name specified by `GHOST`.

If everything went well, Ghost instance should be up and running under
https://ghost.swarm.chbk.co 🎉

### (Optional) Portainer

If you fancy some GUI to look at your awesome containers, give [Portainer](https://www.portainer.io) 
a try. The stack deploys two services: one is a global one called
`portainer_agent` that goes to every single instance in the cluster to gather
data about it and the second one is stateful, i.e. it's gonna require volume to
run properly.

Set a label on one of the manager nodes to handle data:

```
docker node update --label-add portainer-data=true swarm-lb-1
```

Rollout the stack:

```
docker stack deploy -c swarm/portainer.yml portainer
```

If everything went as it shoud, Portainer instance will be available under
https://portainer.swarm.chbk.co.

