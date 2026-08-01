---
title: local k8s with k0s
toc: true
header-includes:
    <link rel="stylesheet" href="/style.css">
---

# About this page

Sometimes you want to test out how an application behaves in kubernetes instead of bare metal.  
This page documents my method of setting up a local k8s cluster quickly for testing purposes.



# Install docker

We need docker to build containers.  
This is the debian installation commands copied over from the above docker page for ease of use.

Source: [https://docs.docker.com/engine/install/debian/](https://docs.docker.com/engine/install/debian/)

Copied over on 2024.09.29

```bash
# Add Docker's official GPG key:
sudo apt-get update
sudo apt-get install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/debian/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# Add the repository to Apt sources:
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/debian \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```



# Create k8s cluster with K0s

For installing the cluster itself and starting/stopping it I use k0s.  
It is fast, easy to use and works out of the box on all servers I had. (debian11,debian12,rocky9)

Documentation: [https://docs.k0sproject.io/stable/install/](https://docs.k0sproject.io/stable/install/)

```bash
# get k0s
curl -sSLf https://get.k0s.sh | sudo sh

# install
sudo k0s install controller --single
sudo k0s start

# kubectl can be used from k0s
k0s kubectl get nodes
# or generate kubeconfig
k0s kubeconfig admin > .kube/config
chmod go-rwx .kube/config

# stop cluster
sudo k0s stop

# delete cluster
sudo k0s reset
sudo reboot
```



# Create local container registry

Start your local registry:

```bash
docker run -d -p 5000:5000 --restart always --name registry registry:2
```

Start a simple frontend for your local registry with (if your server ip is 192.168.1.2):

```bash
sudo docker run -d \
  -e ENV_DOCKER_REGISTRY_HOST=192.168.1.2 \
  -e ENV_DOCKER_REGISTRY_PORT=5000 \
  -p 5001:80 konradkleine/docker-registry-frontend:v2
```

You can now head to to your servers port 5001 with your browser (e.g. http://192.168.1.2:5001/) and see all the containers you pushed.



# Create example container and deployment

For example: You want to create an application that outputs 23.000 random bytes to stdout every second. (To test logging or similar)

Create a `Dockerfile` file with the following inside:

```bash
FROM debian:latest
CMD ["/bin/sh","-c","while true; do tr -dc A-Za-z0-9 </dev/random | head -c 23000; echo; sleep 1; done"]
```

This section expects that you started your local container registry.  
Your kubernetes cluster has to pull the container from somewhere, so for your deployment to work you have to push your built container to your registry.  

Build it with:

```bash
docker build . -t localhost:5000/vomitlogs
```

Push it to your local registry with:

```bash
docker push --tls-verify=false localhost:5000/vomitlogs
```

The tls-verify flag is needed because docker push defaults to HTTPS, which is not enabled here. (Thanks Malik!)



Now you can create a `deployment.yaml` file with the following inside:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: vomitlogs-d
  labels:
    app: vomitlogs
spec:
  replicas: 2
  selector:
    matchLabels:
      app: vomitlogs
  template:
    metadata:
      labels:
        app: vomitlogs
    spec:
      containers:
      - name: vomitlogs
        image: localhost:5000/vomitlogs
```

To now deploy this inside your cluster use:

```bash
k0s kubectl apply -f deployment.yaml
```



# Create persistent volume

Create a folder for your volume:

```bash
mkdir /opt/pv2
```

Create a file named `pv2g.yaml` with the following inside:

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv2
spec:
  capacity:
    storage: 2Gi
  volumeMode: Filesystem
  accessModes:
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Delete
  local:
    path: /opt/pv2
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - localhost.localdomain
```

Run the following to create your PersistentVolume:

```bash
k0s kubectl apply -f pv2g.yaml
```

# Installing argocd

This is just a cheat sheet for installing argocd quickly.

Docs: [https://argo-cd.readthedocs.io/en/stable/getting_started/](https://argo-cd.readthedocs.io/en/stable/getting_started/)

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath='{.data.password}' | base64 -d

kubectl port-forward svc/argocd-server --address 0.0.0.0 -n argocd 8081:443
```
