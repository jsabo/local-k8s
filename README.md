# Local Clusters for Kubernetes Testing

## Run Kubernetes IN Docker

```
cat <<EOF | kind create cluster --name blue --config=-
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
  - role: control-plane
  - role: worker
networking:
  podSubnet: "10.5.0.0/24"

EOF
cat <<EOF | kind create cluster --name green --config=-
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
  - role: control-plane
  - role: worker
networking:
  podSubnet: "10.6.0.0/24"
EOF
```

OR 

## Run Talos Kubernetes in Docker

```
talosctl cluster create --name blue --cidr 10.5.0.0/24
talosctl cluster create --name green --cidr 10.6.0.0/24
```

## Install Stars Demo App

```
kubectl --context admin@green apply -f apps
kubectl --context admin@blue apply -f apps
kubectl --context admin@green get service management-ui -n management-ui -o=jsonpath='{.spec.ports[0].nodePort}'
kubectl --context admin@blue get service management-ui -n management-ui -o=jsonpath='{.spec.ports[0].nodePort}'
```
