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

## Job

```bash
cat <<EOF > backup-job.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: backup
spec:
  template:
    spec:
      containers:
      - name: backup
        image: busybox:1.37.0
        command: ["/bin/sh",  "-c", "cp /etc/hostname /mnt/hostname-$(date +%s).txt"]
        volumeMounts:
        - name: data
          mountPath: /mnt
      restartPolicy: Never
      volumes:
      - name: data
        hostPath:
          path: /data/backup
          type: DirectoryOrCreate
  backoffLimit: 3
EOF
kubectl apply -f backup-job.yaml

kubectl get pods -o wide

# On worker1 or worker2
ls /data/backup
```

## CronJob

```bash
cat <<EOF > backup-cronjob.yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  creationTimestamp: null
  name: backup
spec:
  jobTemplate:
    metadata:
      creationTimestamp: null
      name: backup
    spec:
      template:
        metadata:
          creationTimestamp: null
        spec:
          restartPolicy: Never
          containers:
          - image: busybox:1.37.0
            name: backup
            command:
              - /bin/sh
              - -c
              - cp /etc/hostname /mnt/hostname
            volumeMounts:
              - name: data
                mountPath: /mnt
          volumes:
            - name: data
              hostPath:
                path: /data/asdasd
                type: DirectoryOrCreate
  schedule: '* * * * *'
EOF
kubectl apply -f backup-cronjob.yaml

kubectl get pods -o wide

# On worker1 or worker2
ls /data/backup

kubectl delete cj backup
```

### Review

- Create CronJob named `backup-kubenesia-com` to backup https://kubenesia.com every 5 minutes. File should be visible on worker with location `/data/kubenesia-backup/index.html`.
- Simulate a scheduled database backup:
  - Create a `postgres` StatefulSet with database `test_db` and a single table named `users`.
  - Create a Service named `postgres`.
  - Create a CronJob named `postgres-backup` that run `pg_dump -h postgres -U postgres test_db -f /mnt/test_db.sql`. Use environment variable `PGPASSWORD`.
  - The backup file should be visible on worker node on `/data/postgres-backup/test_db.sql`.
  - Backup should be updated every hour.

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
        - name: logshipper
          image: alpine:latest
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
    image: busybox:1.37
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
/srv/nfs4 10.0.0.0/8(rw,no_subtree_check,no_root_squash)
EOF
sudo mkdir /srv/nfs4
sudo chmod -R 777 /srv/nfs4
sudo systemctl restart nfs-kernel-server
```

### Install NFS client (worker)

```bash
sudo apt-get install --yes nfs-common
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

## Canary deployment

```bash
cat <<EOF >nginx-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx
spec:
  type: ClusterIP
  selector:
    app: nginx
  ports:
  - port: 80
    targetPort: 80
EOF
kubectl apply -f nginx-service.yaml

cat <<EOF >nginx-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
spec:
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.27.1
EOF
kubectl apply -f nginx-deployment.yaml

kubectl run --rm -ti --image=nicolaka/netshoot -- bash
curl -Is nginx | grep Server
exit

cat <<EOF >nginx-canary-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-canary
spec:
  selector:
    matchLabels:
      app: nginx
      canary: "true"
  template:
    metadata:
      labels:
        app: nginx
        canary: "true"
    spec:
      containers:
      - name: nginx
        image: nginx:1.27.2
EOF
kubectl apply -f nginx-canary-deployment.yaml

kubectl run --rm -ti --image=nicolaka/netshoot -- bash
for i in {1..10}; do curl -Is nginx | grep Server; done
exit

kubectl delete deployment nginx
kubectl delete deployment nginx-canary
```

## Rolling update

````bash
cat <<EOF >kubeapp.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kubeapp
  labels:
    app: kubeapp
spec:
  replicas: 5
  selector:
    matchLabels:
      app: kubeapp
  template:
    metadata:
      labels:
        app: kubeapp
    spec:
      containers:
      - name: kubeapp
        image: kubenesia/kubeapp:1.0.0
EOF

kubectl apply -f kubeapp.yaml
kubectl get pods

kubectl set image deployment kubeapp kubeapp=kubenesia/kubeapp:1.1.0
kubectl get pods

```bash
cat <<EOF >kubeapp.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: kubeapp
  name: kubeapp
spec:
  replicas: 4
  selector:
    matchLabels:
      app: kubeapp
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 50%
  template:
    metadata:
      labels:
        app: kubeapp
    spec:
      containers:
        - image: kubenesia/kubeapp:1.2.0
          name: kubeapp
EOF
kubectl apply -f kubeapp.yaml
kubectl get pods
````

## Recreate update

```bash
cat <<EOF >kubeapp.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: kubeapp
  name: kubeapp
spec:
  replicas: 4
  selector:
    matchLabels:
      app: kubeapp
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: kubeapp
    spec:
      containers:
        - image: kubenesia/kubeapp:1.0.0
          name: kubeapp
EOF

kubectl apply -f kubeapp.yaml
kubectl get pods

kubectl set image deployment kubeapp kubeapp=kubenesia/kubeapp:1.1.0
kubectl get pods
```

## Helm

### Install

```bash
sudo apt-get install git yq -y
curl -fsSL https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
which helm
helm version
```

### Create chart

```bash
helm create chart-test
cd chart-test
ls templates
cat values.yaml
```

### Deploy chart using custom values

```bash
cat <<EOF >/tmp/custom-values.yaml
replicaCount: 2
image:
  repository: nginx
  tag: 1.27.2
EOF
helm install nginx . -f /tmp/custom-values.yaml

helm ls
kubectl get pods
kubectl get svc

helm uninstall nginx
```

### Deploy existing chart using custom values

```bash
cat <<EOF >/tmp/blog-values.yaml
wordpressUsername: robot
wordpressPassword: P@ssw0rd
wordpressEmail: robot@gmail.com
wordpressFirstName: Robot
wordpressLastName: Robot
wordpressBlogName: Blog of Robot
persistence:
  enabled: false
mariadb:
  primary:
    persistence:
      enabled: false
EOF
helm install blog oci://registry-1.docker.io/bitnamicharts/wordpress -f /tmp/blog-values.yaml

kubectl get pods
export NODE_PORT=$(kubectl get svc blog-wordpress -o yaml | yq .spec.ports[0].nodePort)
echo $NODE_PORT
curl localhost:$NODE_PORT

helm uninstall blog
```

## Kustomize

```bash
mkdir ~/nginx-kustomize
cd ~/nginx-kustomize
mkdir base development production

cat <<EOF >base/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx
spec:
  type: ClusterIP
  selector:
    app: nginx
  ports:
  - port: 80
    targetPort: 80
EOF

cat <<EOF >base/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
spec:
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.27.1
EOF

cat <<EOF >base/kustomization.yaml
resources:
- service.yaml
- deployment.yaml
EOF

cat <<EOF >development/kustomization.yaml
resources:
- ../base
patches:
- target:
    group: apps
    version: v1
    kind: Deployment
    name: nginx
  patch: |-
    - op: replace
      path: /spec/replicas
      value: 2
EOF

cat <<EOF >production/kustomization.yaml
resources:
- ../base
patches:
- target:
    group: apps
    version: v1
    kind: Deployment
    name: nginx
  patch: |-
    - op: replace
      path: /spec/replicas
      value: 5
EOF

kubectl kustomize development
kubectl kustomize production
```

# Application Observability and Maintenance

## API deprecations

```bash
kubectl api-versions

cat <<EOF >nginx-old-deployment.yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: nginx-old
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx-old
  template:
    metadata:
      labels:
        app: nginx-old
    spec:
      containers:
      - name: nginx
        image: nginx:1.27.2
EOF
vim nginx-old-deployment.yaml # fix the apiVersion
kubectl apply -f nginx-old-deployment.yaml
```

### Startup Probe

Used to protect slow starting containers from serving traffic when they are not ready.

```bash
cat <<EOF >kuard.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: kuard
  name: kuard
spec:
  replicas: 1
  selector:
    matchLabels:
      app: kuard
  template:
    metadata:
      labels:
        app: kuard
    spec:
      containers:
      - image: gcr.io/kuar-demo/kuard-amd64:blue
        name: kuard-amd64
        startupProbe:
          httpGet:
            path: /ready
            port: 6000
EOF

kubectl apply -f kuard.yaml
kubectl get pods
kubectl describe pods -l app=kuard

sed -i 's/6000/8080/g' kuard.yaml
kubectl apply -f kuard.yaml
kubectl describe pods -l app=kuard
```

### Readiness Probe

Used to check the health of the Pod periodically. Pod will be not be restarted if fail, only excluded from traffic serving.

```bash
cat <<EOF >kuard.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: kuard
  name: kuard
spec:
  replicas: 1
  selector:
    matchLabels:
      app: kuard
  template:
    metadata:
      labels:
        app: kuard
    spec:
      containers:
      - image: gcr.io/kuar-demo/kuard-amd64:blue
        name: kuard-amd64
        startupProbe:
          httpGet:
            path: /ready
            port: 8080
        readinessProbe:
          httpGet:
            path: /healthy
            port: 8080
EOF

kubectl apply -f kuard.yaml
kubectl get pods
kubectl describe pods -l app=kuard
```

### Liveness Probe

Used to check the health of the Pod periodically. Pod will be restarted if fail.

```bash
cat <<EOF >kuard.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: kuard
  name: kuard
spec:
  replicas: 1
  selector:
    matchLabels:
      app: kuard
  template:
    metadata:
      labels:
        app: kuard
    spec:
      containers:
      - image: gcr.io/kuar-demo/kuard-amd64:blue
        name: kuard-amd64
        startupProbe:
          httpGet:
            path: /ready
            port: 8080
        livenessProbe:
          httpGet:
            path: /healthy
            port: 8080
EOF

kubectl apply -f kuard.yaml
kubectl get pods
kubectl describe pods -l app=kuard
```

### Review

- Try to create a Deployment with image `gcr.io/kuar-demo/kuard-amd64:blue` with 5 replicas and access port 8080 of the container.
- Try to create a Deployment with the following spec:
  - Name: `web`
  - Two images: `nginx:1.27.2` and `gcr.io/kuar-demo/kuard-amd64:blue`
  - Replica: 10
  - Startup probe to nginx
  - Liveness probe to kuard

## Inspect container logs

```bash
kubectl logs deployment/kuard
kubectl logs -f deployment/kuard
```

# Application Environment, Configuration and Security

## Requests, limits, and quotas

- Used to control the resource usage (CPU and memory) of a Pod.
- Best practice: do not use CPU limit to prevent throttling: (https://home.robusta.dev/blog/stop-using-cpu-limits)
- Always define resource requests
  - Easier capacity planning
  - Less surprise

### Install metrics-server

```bash
curl -fsSLo metrics-server.yaml https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
vim metrics-server.yaml
# add --kubelet-insecure-tls under args
kubectl apply -f metrics-server.yaml
kubectl -n kube-system get pods
kubectl top nodes
kubectl top pods
```

### Create Pod with resource requests and limits

```bash
cat <<EOF >pod-requests-limits.yaml
apiVersion: v1
kind: Pod
metadata:
  name: kubeapp-requests-limits
spec:
  containers:
  - name: kubeapp
    image: kubenesia/kubeapp
    resources:
      requests:
        memory: 128Mi
        cpu: 250m
      limits:
        memory: 256Mi
        cpu: 500m
EOF

kubectl apply -f pod-requests-limits.yaml

kubectl get pods
kubectl describe pods

kubectl describe node worker1
kubectl describe node worker2
```

### Create Pod with bad resource requests and limits

- If resource requests can't be fulfilled by any node, then Pod will stuck in Pending state.

```bash
cat <<EOF >pod-requests-limits.yaml
apiVersion: v1
kind: Pod
metadata:
  name: kubeapp-requests-limits
spec:
  containers:
  - name: kubeapp
    image: kubenesia/kubeapp:1.2.0
    resources:
      requests:
        memory: 128Gi
        cpu: 250m
      limits:
        memory: 256Gi
        cpu: 500m
EOF

kubectl apply -f pod-requests-limits.yaml

kubectl get pods
kubectl describe pods

kubectl describe node worker1
kubectl describe node worker2

kubectl delete pod kubeapp-requests-limits
```

## ConfigMap

### Create ConfigMap

```bash
cat <<EOF >kubeapp-configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: kubeapp
data:
  MODE: development
  COLOR: blue
  SERVICE: kubeapp
EOF

kubectl apply -f kubeapp-configmap.yaml
kubectl get cm kubeapp
kubectl get cm kubeapp -o yaml
```

### ConfigMap as env variable

```bash
cat <<EOF >kubeapp-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: kubeapp
  name: kubeapp
spec:
  replicas: 1
  selector:
    matchLabels:
      app: kubeapp
  template:
    metadata:
      labels:
        app: kubeapp
    spec:
      containers:
      - image: kubenesia/kubeapp:1.2.0
        name: kubeapp
        envFrom:
          - configMapRef:
              name: kubeapp
EOF

kubectl apply -f kubeapp-deployment.yaml
kubectl exec -ti deployment/kubeapp -- sh
printenv | egrep 'COLOR=|MODE=|SERVICE='
exit
```

### ConfigMap as file

```bash
cat <<EOF >kubeapp-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: kubeapp
  name: kubeapp
spec:
  replicas: 1
  selector:
    matchLabels:
      app: kubeapp
  template:
    metadata:
      labels:
        app: kubeapp
    spec:
      containers:
      - image: kubenesia/kubeapp:1.2.0
        name: kubeapp
        volumeMounts:
          - mountPath: /data/config
            name: config
      volumes:
        - name: config
          configMap:
            name: kubeapp
EOF

kubectl apply -f kubeapp-deployment.yaml
kubectl exec -ti deployment/kubeapp -- sh
ls /data/config
cat /data/config
exit
```

## Secret

### Create Secret

```bash
kubectl create secret generic kubeapp --from-literal=MODE=development --from-literal=COLOR=blue
kubectl get secret
kubectl get secret kubeapp -o yaml
```

### Secret as env variable

```bash
kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: kubeapp
  name: kubeapp
spec:
  replicas: 1
  selector:
    matchLabels:
      app: kubeapp
  template:
    metadata:
      labels:
        app: kubeapp
    spec:
      containers:
      - image: kubenesia/kubeapp:1.2.0
        name: kubeapp
        envFrom:
          - secretRef:
              name: kubeapp
EOF

kubectl exec -ti deployment/kubeapp -- sh
printenv | egrep 'COLOR=|MODE=|SERVICE='
exit
```

### Secret as file

```bash
kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: kubeapp
  name: kubeapp
spec:
  replicas: 1
  selector:
    matchLabels:
      app: kubeapp
  template:
    metadata:
      labels:
        app: kubeapp
    spec:
      containers:
      - image: kubenesia/kubeapp:1.2.0
        name: kubeapp
        volumeMounts:
          - mountPath: /data/config
            name: config
      volumes:
        - name: config
          configMap:
            name: kubeapp
EOF

kubectl exec -ti deployment/kubeapp -- sh
ls /data/config
cat /data/config
exit
```

### Review

- Create a `postgres` StatefulSet as before, but use a `Secret` to configure `POSTGRES_PASSWORD`.
- Check with `kubectl exec -ti postgres-0 -- psql -h 127.0.0.1 -U postgres`

## ServiceAccount


### Create ServiceAccount
```bash
kubectl create serviceaccount kubeapp
kubectl get serviceaccount kubeapp
```

### Use ServiceAccount on Pod
```bash
cat <<EOF >kubeapp-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kubeapp
  labels:
    app: kubeapp
spec:
  replicas: 2
  selector:
    matchLabels:
      app: kubeapp
  template:
    metadata:
      labels:
        app: kubeapp
    spec:
      serviceAccountName: kubeapp
      containers:
        - name: kubeapp
          image: kubenesia/kubeapp:1.2.0
          ports:
            - name: http
              containerPort: 8000
EOF
kubectl apply -f kubeapp-deployment.yaml
kubectl get pods
kubectl describe pods
```

## SecurityContext

```bash
cat <<EOF >node-hello-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: node-hello
spec:
  containers:
  - name: node-hello
    image: gcr.io/google-samples/node-hello:1.0
EOF
kubectl apply -f node-hello-pod.yaml
kubectl exec -ti node-hello -- bash
whoami
touch test.txt
ls -ls test.txt
exit

cat <<EOF >node-hello-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: node-hello
spec:
  containers:
  - name: node-hello
    image: gcr.io/google-samples/node-hello:1.0
    securityContext:
      runAsUser: 1000
      allowPrivilegeEscalation: false
EOF
kubectl apply -f node-hello-pod.yaml --force
kubectl exec -ti node-hello -- bash
whoami
touch test.txt
exit

kubectl delete -f node-hello-pod.yaml
```

# Services & Networking

- Containers inside the same pod can communicate via `localhost`.
- Each pod in a cluster gets its own unique IP address.
- All pods can communicate with all other pods by default.

## Service

### ClusterIP

- ClusterIP provides a stable IP address or hostname instead of manually using IP of pods.

```bash
cat <<EOF >kubeapp-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kubeapp
  labels:
    app: kubeapp
spec:
  replicas: 2
  selector:
    matchLabels:
      app: kubeapp
  template:
    metadata:
      labels:
        app: kubeapp
    spec:
      containers:
        - name: kubeapp
          image: kubenesia/kubeapp:1.2.0
          ports:
            - name: http
              containerPort: 8000
EOF

cat <<EOF >kubeapp-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: kubeapp
  labels:
    app: kubeapp
spec:
  type: ClusterIP
  selector:
    app: kubeapp
  ports:
    - port: 80
      protocol: TCP
      targetPort: 8000
EOF

kubectl apply -f kubeapp-deployment.yaml -f kubeapp-service.yaml
kubectl get svc
kubectl describe svc kubeapp
kubectl get ep
kubectl describe ep
kubectl get pods -o wide
kubectl get pods -l app=kubeapp

kubectl run netshoot -ti --rm --image=nicolaka/netshoot
for i in {1..9}; do curl kubeapp; done
nslookup kubeapp
exit
```

### Headless Service

- Used to bypass the cluster-wide IP address, name will be resolved directly to pod IPs.
- Load balancing logic will be responsibility of the client.

```bash
cat <<EOF >kubeapp-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: kubeapp
spec:
  type: ClusterIP
  clusterIP: None
  selector:
    app: kubeapp
  ports:
    - port: 80
      protocol: TCP
      targetPort: 8000
EOF

kubectl apply -f kubeapp-service.yaml
kubectl get svc
kubectl describe svc kubeapp

kubectl run test -it --rm --image=kubenesia/kubebox -- sh
curl kubeapp
nslookup kubeapp
exit
```

### ExternalName

- Create a DNS mapping for external hostname.

```bash
cat <<EOF >get-ip-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: get-ip
spec:
  type: ExternalName
  externalName: google.com
EOF

kubectl apply -f get-ip-service.yaml
kubectl get svc
kubectl describe svc get-ip

kubectl run netshoot -it --rm --image=nicolaka/netshoot
curl get-ip
nslookup get-ip
exit
```

### NodePort

- Provide a static port that is externally accessible on all nodes.

```bash
cat <<EOF >kubeapp-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: kubeapp
spec:
  type: NodePort
  selector:
    app: kubeapp
  ports:
    - port: 80
      protocol: TCP
      targetPort: 8000
EOF

kubectl apply -f kubeapp-service.yaml
kubectl get svc
kubectl describe svc kubeapp

export PUBLIC_IP=$(curl -s icanhazip.com)
export NODE_PORT=$(kubectl get svc kubeapp-service -o yaml | yq '.spec.ports[0].nodePort')
curl "$PUBLIC_IP:$NODE_PORT"
```

## Ingress

- Route HTTP/HTTPS traffic into cluster workloads.

### Install ingress controller

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.12.0-beta.0/deploy/static/provider/baremetal/deploy.yaml
kubectl get pods -n ingress-nginx
kubectl get svc -n ingress-nginx
```

### Create Deployment & Service

```bash
kubectl create deployment blue --image=kubenesia/kubeapp:1.2.0
kubectl create deployment green --image=kubenesia/kubeapp:1.2.0

kubectl expose deployment blue --port=80 --target-port=8000
kubectl expose deployment green --port=80 --target-port=8000
```

### Route by host header

```bash
kubectl create ingress blue --class=nginx --rule="blue.$PUBLIC_IP.sslip.io/*=blue:80"
kubectl create ingress green --class=nginx --rule="green.$PUBLIC_IP.sslip.io/*=green:80"

kubectl get ingress
kubectl get ingress blue -o yaml
kubectl get ingress green -o yaml

export NODE_PORT=$(kubectl get svc -n ingress-nginx ingress-nginx-controller -o yaml | yq '.spec.ports[0].nodePort')
echo $NODE_PORT

curl http://blue.$PUBLIC_IP.sslip.io:$NODE_PORT
curl http://green.$PUBLIC_IP.sslip.io:$NODE_PORT
```

### Route by path

````bash
cat <<EOF >blue-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: blue
spec:
  ingressClassName: nginx
  rules:
  - http:
      paths:
      - backend:
          service:
            name: blue
            port:
              number: 80
        path: /blue
        pathType: Prefix
EOF

kubectl apply -f blue-ingress.yaml

cat <<EOF >green-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: green
spec:
  ingressClassName: nginx
  rules:
  - http:
      paths:
      - backend:
          service:
            name: green
            port:
              number: 80
        path: /green
        pathType: Prefix
EOF

kubectl apply -f green-ingress.yaml

curl http://$PUBLIC_IP.sslip.io:$NODE_PORT/blue
curl http://$PUBLIC_IP.sslip.io:$NODE_PORT/green
````

## Network Policy

```bash
# Create target workload
kubectl create deployment kubeapp --image=kubenesia/kubeapp:1.2.0 --port=8000
kubectl expose deployment kubeapp --port=80 --target-port=8000

# Create NetworkPolicy to prevent access from other namespace
cat <<EOF >netpol-deny-from-other-ns.yaml
kind: NetworkPolicy
apiVersion: networking.k8s.io/v1
metadata:
  namespace: default
  name: deny-from-other-namespaces
spec:
  podSelector: {}
  ingress:
  - from:
    - podSelector: {}
EOF
kubectl apply -f netpol-deny-from-other-ns.yaml

# Check NetworkPolicy
kubectl get networkpolicy

# Test access from other namespace
kubectl create ns dev
kubectl run netshoot -it --rm --image=nicolaka/netshoot -n dev
wget -qO- --timeout=2 kubeapp.default # failed from different namespace
kubectl run netshoot -it --rm --image=nicolaka/netshoot
wget -qO- --timeout=2 kubeapp.default # success from same namespace

# Remove NetworkPolicy
kubectl delete netpol deny-from-other-namespaces
```
