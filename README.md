# CKAD

CKAD training notes.

# Installation

## Common configuration

### Modify system settings

```bash
# enable ip forwarding
cat <<EOF | sudo tee /etc/sysctl.d/10-ipv4-forward.conf
net.ipv4.ip_forward = 1
EOF

sudo sysctl --system
echo 1 | sudo tee /proc/sys/net/ipv4/ip_forward

# disable memory swap
sudo swapoff -a
```

### Install containerd

```bash
# add tools
sudo apt-get clean
sudo apt-get update
sudo apt-get install apt-transport-https ca-certificates curl gnupg lsb-release -y

# add containerd repository
sudo curl -fsSL https://download.docker.com/linux/debian/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/debian \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# install containerd
sudo apt-get update && sudo apt-get install containerd.io -y
```

### Configure containerd

```bash
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/g' /etc/containerd/config.toml
grep SystemdCgroup /etc/containerd/config.toml
sudo systemctl restart containerd
sudo systemctl is-active containerd
```

### Install kubernetes tools

```bash
# add kubernetes repository
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.30/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.30/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list

# install kubernetes tools
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

## Controller configuration

### Init cluster

```bash
sudo kubeadm init
```

### Configure easy access to cluster

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
source <(kubectl completion bash)
```

### Check cluster

```bash
kubectl get node
kubectl get pods -n kube-system
```

### Generate join token

```bash
kubeadm token create --print-join-command
```

## Worker configuration

### Join cluster

```bash
sudo kubeadm join $CONTROLLER_IP:6443 --token $JOIN_TOKEN --discovery-token-ca-cert-hash $CA_CERT_HASH
```

## CNI installation (controller)

```bash
curl https://raw.githubusercontent.com/projectcalico/calico/v3.28.2/manifests/calico.yaml -O
kubectl apply -f calico.yaml
kubectl get pods -n kube-system | grep calico
```

# Application Design and Build

## Building container images

```bash
mkdir gohello

cat <<EOF >go.mod
module gohello

go 1.23.2
EOF

cat <<EOF >main.go
package main

import (
    "fmt"
    "net/http"
)

func main() {
    http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
        fmt.Fprintf(w, "Hello, you've requested: %s\n", r.URL.Path)
    })

    http.ListenAndServe(":8080", nil)
}
EOF

cat <<EOF >Dockerfile
FROM golang:1.23.2 AS builder
WORKDIR /app
COPY . .
RUN go build -o gohello .
ENTRYPOINT [ "./gohello" ]
EOF

docker login
docker build -t $DOCKERHUB_USERNAME/gohello:1.0.0
docker push $DOCKERHUB_USERNAME/gohello:1.0.0

cat <<EOF >Dockerfile
FROM golang:1.23.2 AS builder
WORKDIR /app
COPY . .
RUN go build -o gohello .

FROM gcr.io/distroless/base-debian12
WORKDIR /app
COPY --from=builder /app/gohello .
ENTRYPOINT [ "./gohello" ]
EOF

docker login
docker build -t $DOCKERHUB_USERNAME/gohello:1.1.0
docker push $DOCKERHUB_USERNAME/gohello:1.1.0
```

## Deployment

### Create Deployment

```bash
kubectl create deployment nginx --image=nginx:1.27.2 --output=yaml --dry-run=client >nginx-deployment.yaml
cat nginx-deployment.yaml
kubectl apply -f nginx-deployment.yaml

kubectl get deployment
kubectl get deployment nginx
kuebctl get deployment nginx -o yaml
kubectl describe deployment nginx
kubectl get pods

kubectl port-forward deployment/nginx --address=0.0.0.0 8080:80
curl -v localhost:8080 # or access public ip via browser
```

### Update Deployment

```bash
sed -i 's/1.27.2/1.27.1/g' nginx-deployment.yaml
cat nginx-deployment.yaml
kubectl apply -f nginx-deployment.yaml
kubectl get deployment nginx
kubectl get pods

kubectl port-forward deployment/nginx --address=0.0.0.0 8080:80
curl -v localhost:8080 # or access public ip via browser
```

### Scale Deployment

```bash
kubectl scale deployment nginx --replicas=2
kubectl get deployment
kubectl get pods
kubectl edit deployment nginx # set replicas back to 1
```

### Review

- Create a Deployment named `httpd` with image `httpd:2.46.2`. What is the result?

## DaemonSet

Ensure all nodes run a copy of a Pod. Usually used for operational tools or add-ons.

```bash
cat <<EOF >nginx-daemonset.yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  selector:
    matchLabels:
      name: nginx
  template:
    metadata:
      labels:
        name: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.27.2
EOF

kubectl apply -f nginx-daemonset.yaml
```

### Review

- Create a DaemonSet with name `kuard` that runs `gcr.io/kuar-demo/kuard-amd64:blue` image.

## StatefulSet

Like Deployment, but used to deploy stateful application. Maintain a sticky identity for each Pod.

### Create StatefulSet

```bash
cat <<EOF >mysql-statefulset.yaml
apiVersion: v1
kind: Service
metadata:
  name: mysql
  labels:
    app: mysql
spec:
  ports:
    - port: 3306
      name: mysql
  clusterIP: None
  selector:
    app: mysql
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
  labels:
    app: mysql
spec:
  serviceName: mysql
  replicas: 1
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
      - name: mysql
        image: mysql:8.4.2
        ports:
        - containerPort: 3306
          name: mysql
        env:
        - name: MYSQL_ROOT_PASSWORD
          value: "P@ssw0rd"
        - name: MYSQL_DATABASE
          value: "mydb"
        - name: MYSQL_USER
          value: "user"
        - name: MYSQL_PASSWORD
          value: "P@ssw0rd"
        volumeMounts:
        - name: data
          mountPath: /var/lib/mysql
      volumes:
      - name: data
        hostPath:
          path: /data/mysql
          type: DirectoryOrCreate
EOF

kubectl apply -f mysql-statefulset.yaml
kubectl get sts -o wide
```

### Create database

```bash
kubectl exec -ti mysql-0 -- bash
mysql -h 127.0.0.1 -u root -p mydb
select version();
create table users (id int primary key, email text);
show tables;
exit
```

### Modify database version

```bash
sed -i 's/8.4.2/8.4.3/g' mysql-statefulset.yaml
kubectl apply -f mysql-statefulset.yaml
kubectl exec -ti mysql-0 -- bash
mysql -h 127.0.0.1 -u root -p mydb
select version();
show tables;
```

### Review

- Try to create a StatefulSet for postgres.
- Check [here](https://hub.docker.com/_/postgres) for information related to the container image.

## Sidecar container

```bash
cat <<EOF >myapp-with-sidecar.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  labels:
    app: myapp
spec:
  replicas: 1
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
        - name: myapp
          image: alpine:latest
          command: ['sh', '-c', 'while true; do echo "logging" >> /opt/logs.txt; sleep 1; done']
          volumeMounts:
            - name: data
              mountPath: /opt
      initContainers:
        - name: logshipper
          image: alpine:latest
          restartPolicy: Always
          command: ['sh', '-c', 'tail -F /opt/logs.txt']
          volumeMounts:
            - name: data
              mountPath: /opt
      volumes:
        - name: data
          emptyDir: {}
EOF

kubectl apply -f sidecar.yaml
kubectl get pods

kubectl logs myapp -c logshipper
```

## Init container

```bash
cat <<EOF >nginx-with-init-container-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-with-init-container
spec:
  containers:
  - name: nginx
    image: nginx
    volumeMounts:
    - name: html
      mountPath: /usr/share/nginx/html
  initContainers:
  - name: install
    image: busybox:1.28
    command:
    - wget
    - -O
    - /usr/share/nginx/html/index.html
    - https://kubenesia.com
    volumeMounts:
    - name: html
      mountPath: /usr/share/nginx/html
  volumes:
  - name: html
    emptyDir: {}
EOF
kubectl apply -f nginx-with-init-container-pod.yaml
kubectl get pods nginx-with-init-container
kubectl port-forward nginx-with-init-container 8080:80
curl -v localhost:8080
```

## PersistentVolume

### Set up NFS server

```bash
sudo apt-get install --yes nfs-kernel-server
cat <<EOF | sudo tee /etc/exports
/srv/nfs4 10.0.0.0/8(rw,no_subtree_check,all_squash)
EOF
sudo systemctl restart nfs-kernel-server
sudo mkdir /srv/nfs4
sudo chown 65534:65534 /srv/nfs4
```

### Static provisioning

```bash
export NFS_SERVER=$(ip -4 a | grep global | head -n 1 | awk '{print $2}' | cut -d '/' -f 1)

cat <<EOF >nginx-pv.yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: nginx
spec:
  storageClassName: "nfs"
  capacity:
    storage: 1Gi
  accessModes:
   - ReadWriteMany
  nfs:
    server: $NFS_SERVER
    path: /srv/nfs4
EOF

kubectl apply -f nginx-pv.yaml

kubectl get pv
kubectl describe pv nginx

cat <<EOF >nginx-pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: nginx
spec:
  storageClassName: "nfs"
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 1Gi
EOF

kubectl apply -f nginx-pvc.yaml
kubectl get pvc
kubectl describe pvc nginx
```

### Mount volume on Pod

```bash
cat <<EOF >nginx-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
  - name: nginx
    image: nginx:1.27.2
    volumeMounts:
    - name: html
      mountPath: /usr/share/nginx/html
  volumes:
  - name: html
    persistentVolumeClaim:
      claimName: nginx
EOF

kubectl apply -f nginx-pod.yaml
kubectl describe pods nginx
kubectl get pods nginx -o wide

kubectl port-forward nginx 8080:80

echo "Old landing page" | sudo tee /srv/nfs4/index.html
curl localhost:8080

echo "New landing page" | sudo tee /srv/nfs4/index.html
curl localhost:8080

kubectl delete pods nginx
kubectl delete pvc nginx
kubectl delete pv nginx
```

### Review

- Create a StatefulSet named `postgres` that uses a PV and a PVC with the same name.
- The PV should be backed by NFS.
- Create a new database with name `test_db` and then remove the pod.
- Check if the pod is recreated and `test_db` still exists.

# Application Deployment

## Blue/Green deployment

## Canary deployment

## Rolling update

## Recreate update

## Helm

## Kustomize

# Application Observability and Maintenance

## API deprecations

## Startup Probe

## Readiness Probe

## Liveness Probe

## Inspect container logs

# Application Environment, Configuration and Security

## Requests, limits, and quotas

## ConfigMap

## Secret

## ServiceAccount

## SecurityContext

# Services and Networking

## ClusterIP

## NodePort

## NetworkPolicy

## Ingress
