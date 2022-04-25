# Docker Swarm Mode setup

After initial playbook is being rolled out, you can setup the swarm. I prefer
running all the things from my local machine instead of SSHing to the manager,
then to the worker etc. etc. So, assuming you've got `docker` command at your
disposal:

```bash
docker context create --docker host=ssh://root@swarm-lb.chbk.co --description "Docker Swarm Manager" swarm
docker context use swarm
docker info
```

The same can be done for the worker:

```bash
docker context create --docker host=ssh://root@swarm-wrk-1.chbk.co --description "Docker Swarm Worker" wrk-1
docker context use wrk-1
docker info
```

Switching between the two is now as easy as `docker context use $NAME`. All the
commands are running via SSH tunnel. On the manager node:

```bash
docker swarm init --advertise-addr eth1
```

(I'm using `--advertise-addr eth1` to establish communication in DO VPC).

This command will spit out `docker swarm join[...]` command with the token that
can be then/later used on the worker to join the cluster.