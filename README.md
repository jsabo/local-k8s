# Local Clusters for Kubernetes Testing

## Creating Talos in Docker Kubernetes clusters

### Install the Talos CLI

```
brew install siderolabs/tap/talosctl
```

### Create the clusters

```
talosctl cluster create
talosctl cluster create --name calico --cidr 10.5.0.0/24
talosctl cluster create --name cilium --cidr 10.6.0.0/24
```

### FAQ

#### How do you find the names of local Talos clusters?

`docker ps` should give you an idea, as the container names are prefix with cluster names. `docker network ls` might also be helpful, as each cluster has its own network. But please keep in mind that there might be other unrelated containers/networks.

#### My local Talos cluster stopped working when Orbstack/Docker Desktop was shutdown.  How do I turn it back on?

Get the container ids for each cluster and start them with `docker`. 

```
$ docker ps -a
CONTAINER ID   IMAGE                             COMMAND        CREATED      STATUS                        PORTS     NAMES
e01809686f0e   ghcr.io/siderolabs/talos:v1.7.5   "/sbin/init"   4 days ago   Exited (137) 24 seconds ago             blue-worker-1
ea2fe178d0b0   ghcr.io/siderolabs/talos:v1.7.5   "/sbin/init"   4 days ago   Exited (137) 24 seconds ago             blue-controlplane-1

$ docker start e01809686f0e ea2fe178d0b0
e01809686f0e
ea2fe178d0b0

$ docker ps
CONTAINER ID   IMAGE                             COMMAND        CREATED      STATUS         PORTS                                               NAMES
e01809686f0e   ghcr.io/siderolabs/talos:v1.7.5   "/sbin/init"   4 days ago   Up 7 seconds                                                       blue-worker-1
ea2fe178d0b0   ghcr.io/siderolabs/talos:v1.7.5   "/sbin/init"   4 days ago   Up 7 seconds   0.0.0.0:49198->6443/tcp, 0.0.0.0:49199->50000/tcp   blue-controlplane-1
```
