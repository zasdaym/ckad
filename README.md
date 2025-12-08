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
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
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
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.34/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.34/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list

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
kubectl create deployment nginx --image=public.ecr.aws/docker/library/nginx:1.27.2 --output=yaml --dry-run=client >nginx-deployment.yaml
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

- Create a Deployment named `httpd` with image `public.ecr.aws/docker/library/httpd:2.46.2`. What is the result?

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
        image: public.ecr.aws/docker/library/nginx:1.27.2
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
        image: public.ecr.aws/docker/library/mysql:8.4.2
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
        image: public.ecr.aws/docker/library/busybox:1.37.0
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
cat <<EOF >echo.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: echo
  labels:
    app: echo
spec:
  replicas: 5
  selector:
    matchLabels:
      app: echo
  template:
    metadata:
      labels:
        app: echo
    spec:
      containers:
      - name: echo
        image: mendhak/http-https-echo:31
EOF

kubectl apply -f echo.yaml
kubectl get pods

kubectl set image deployment echo echo=mendhak-http-https/echo:37
kubectl get pods

```bash
cat <<EOF >echo.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: echo
  name: echo
spec:
  replicas: 4
  selector:
    matchLabels:
      app: echo
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 50%
  template:
    metadata:
      labels:
        app: echo
    spec:
      containers:
        - image: mendhak/http-https-echo:31
          name: echo
EOF
kubectl apply -f echo.yaml
kubectl get pods
````

## Recreate update

```bash
cat <<EOF >echo.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: echo
  name: echo
spec:
  replicas: 4
  selector:
    matchLabels:
      app: echo
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: echo
    spec:
      containers:
        - image: mendhak/http-https-echo:37
          name: echo
EOF

kubectl apply -f echo.yaml
kubectl get pods

kubectl set image deployment echo echo=mendhak/http-https-echo:31
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
            path: /ready
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
  name: echo-requests-limits
spec:
  containers:
  - name: echo
    image: mendhak/http-https-echo:31
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
  name: echo-requests-limits
spec:
  containers:
  - name: echo
    image: mendhak/http-https-echo:31
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

kubectl delete pod echo-requests-limits
```

## ConfigMap

### Create ConfigMap

```bash
cat <<EOF >echo-configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: echo
data:
  MODE: development
  COLOR: blue
  SERVICE: echo
  MULTILINE: |
    Line
    Line
    Line
EOF

kubectl apply -f echo-configmap.yaml
kubectl get cm echo
kubectl get cm echo -o yaml
```

### ConfigMap as env variable

```bash
cat <<EOF >echo-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: echo
  name: echo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: echo
  template:
    metadata:
      labels:
        app: echo
    spec:
      containers:
      - image: mendhak/http-https-echo:31
        name: echo
        envFrom:
          - configMapRef:
              name: echo
EOF

kubectl apply -f echo-deployment.yaml
kubectl exec -ti deployment/echo -- sh
printenv | egrep 'COLOR=|MODE=|SERVICE='
exit
```

### ConfigMap as file

```bash
cat <<EOF >echo-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: echo
  name: echo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: echo
  template:
    metadata:
      labels:
        app: echo
    spec:
      containers:
      - image: mendhak/http-https-echo:31
        name: echo
        volumeMounts:
          - mountPath: /data/config
            name: config
      volumes:
        - name: config
          configMap:
            name: echo
EOF

kubectl apply -f echo-deployment.yaml
kubectl exec -ti deployment/echo -- sh
ls /data/config
cat /data/config
exit
```

## Secret

### Create Secret

```bash
kubectl create secret generic echo --from-literal=MODE=development --from-literal=COLOR=blue
kubectl get secret
kubectl get secret echo -o yaml
```

### Secret as env variable

```bash
kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: echo
  name: echo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: echo
  template:
    metadata:
      labels:
        app: echo
    spec:
      containers:
      - image: mendhak/http-https-echo:31
        name: echo
        envFrom:
          - secretRef:
              name: echo
EOF

kubectl exec -ti deployment/echo -- sh
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
    app: echo
  name: echo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: echo
  template:
    metadata:
      labels:
        app: echo
    spec:
      containers:
      - image: mendhak/http-https-echo:31
        name: echo
        volumeMounts:
          - mountPath: /data/config
            name: config
      volumes:
        - name: config
          configMap:
            name: echo
EOF

kubectl exec -ti deployment/echo -- sh
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
kubectl create serviceaccount pod-reader
kubectl get serviceaccount pod-reader
```

### Create ClusterRole and ClusterRoleBinding
```bash
cat <<EOF >pod-reader-rbac.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: pod-reader
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "watch", "list"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: pod-reader
subjects:
- kind: ServiceAccount
  name: pod-reader
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: pod-reader
EOF
```

### Use ServiceAccount on Pod
```bash
cat <<EOF >pod-reader-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: pod-reader
  labels:
    app: pod-reader
spec:
  replicas: 1
  selector:
    matchLabels:
      app: pod-reader
  template:
    metadata:
      labels:
        app: pod-reader
    spec:
      serviceAccountName: pod-reader
      containers:
        - name: pod-reader
          image: bitnami/kubectl:1.33.4
          command:
            - sleep
            - "3600"
EOF
kubectl apply -f pod-reader-deployment.yaml

kubectl exec -ti deployment/pod-reader -- sh
kubectl get pods
kubectl delete pods --all # failed
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

- Service provides a stable IP address or hostname instead of manually using IP of pods.

### ClusterIP

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

kubectl run test -ti --rm --image=kubenesia/kubebox -- sh
curl kubeapp
nslookup kubeapp
exit
```

### ExternalName

```bash
cat <<EOF >gogolele-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: gogolele
spec:
  type: ExternalName
  externalName: google.com
EOF

kubectl apply -f gogolele.yaml
kubectl get svc
kubectl describe svc get-ip

kubectl run test -it --rm --image=kubenesia/kubebox -- sh
curl gogolele
nslookup gogolele
exit
```

### NodePort

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
      nodePort: 31080
EOF

kubectl apply -f kubeapp-service.yaml
kubectl get svc
kubectl describe svc kubeapp

export WORKER_IP=CHANGE_ME
export NODE_PORT=$(kubectl get svc kubeapp -o yaml | yq '.spec.ports[0].nodePort')
curl "$WORKER_IP:$NODE_PORT"
```

### Headless Service

- Used to bypass the cluster-wide IP address, name will be resolved directly to pod IPs.

```bash
cat <<EOF >kubeapp-headless-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: kubeapp-headless
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

kubectl apply -f kubeapp-headless-service.yaml
kubectl get svc | grep kubeapp
kubectl describe svc kubeapp
kubectl describe svc kubeapp-headless

kubectl run test -it --rm --image=kubenesia/kubebox -- sh

nslookup kubeapp
nslookup kubeapp-headless

curl kubeapp
curl kubeapp-headless:8000
```

### Review

- Run the following commands:

```bash
cat <<EOF >nginx-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx
spec:
  selector:
    application: nginx
    role: web
  ports:
    - port: 80
      protocol: TCP
      targetPort: 8080
EOF
kubectl apply -f nginx-service.yaml
kubectl create deployment nginx --image=nginx:1.27.2
kubectl port-forward svc/nginx --address 0.0.0.0 1234:8080
curl localhost:1234
```

- Does it work? If not, what's wrong?
- Create a Deployment `echo` with image `mendhak/http-https-echo:31`. Create a Service with type `NodePort` to expose the previous Deployment with the same name. The application port is `8080`. Also create a HPA with target CPU utilization of 60% and max replicas of 5.

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
kubectl create deployment blue --image=mendhak/http-https-echo:31
kubectl create deployment green --image=mendhak/http-https-echo:31

kubectl expose deployment blue --port=80 --target-port=8080
kubectl expose deployment green --port=80 --target-port=8080
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

## Gateway

- Gateway API is an official Kubernetes project focused on L4 and L7 routing in Kubernetes.
- Intended to be the next generation of Kubernetes Ingress, Load Balancing, and Service Mesh APIs.
- Designed to be generic, expressive, and role-oriented.

nginx-fabric-gateway will be used here as Gateway API implementation. For other implementations, check here https://gateway-api.sigs.k8s.io/implementations/.

### Install Gateway CRD
```bash
kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.3.0/standard-install.yaml
```

### Install helm
```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```

### Install Gateway controller implementation

```bash
cat <<EOF >/tmp/nginx-gateway.yaml
nginx:
  kind: daemonSet
  service:
    type: NodePort
    nodePorts:
      - port: 30080
        listenerPort: 80
EOF

helm install nginx-gateway oci://ghcr.io/nginx/charts/nginx-gateway-fabric --version 2.0.2 --create-namespace -n nginx-gateway -f /tmp/nginx-gateway.yaml
```

### Create Gateway

```bash
kubectl apply -f- <<EOF
kind: Gateway
apiVersion: gateway.networking.k8s.io/v1
metadata:
  name: nginx-gateway
  namespace: nginx-gateway
spec:
  gatewayClassName: nginx
  listeners:
  - protocol: HTTP
    port: 80
    name: http
    allowedRoutes:
      namespaces:
        from: All
EOF
```

### Route by host header

```bash
kubectl create deployment blue --image=mendhak/http-https-echo:31
kubectl create deployment green --image=mendhak/http-https-echo:31

kubectl expose deployment blue --port=80 --target-port=8080
kubectl expose deployment green --port=80 --target-port=8080

kubectl apply -f - <<EOF
kind: HTTPRoute
apiVersion: gateway.networking.k8s.io/v1
metadata:
  name: blue
  labels:
    app: blue
spec:
  hostnames:
  - blue.example.com
  parentRefs:
  - name: nginx-gateway
    namespace: nginx-gateway
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /
    backendRefs:
    - kind: Service
      name: blue
      port: 80
EOF

kubectl apply -f - <<EOF
kind: HTTPRoute
apiVersion: gateway.networking.k8s.io/v1
metadata:
  name: green
  labels:
    app: green
spec:
  hostnames:
  - green.example.com
  parentRefs:
  - name: nginx-gateway
    namespace: nginx-gateway
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /
    backendRefs:
    - kind: Service
      name: green
      port: 80
EOF

curl --connect-to ::$WORKER_IP:30080 blue.example.com
curl --connect-to ::$WORKER_IP:30080 green.example.com
```

### Route by path

```bash
kubectl apply -f - <<EOF
kind: HTTPRoute
apiVersion: gateway.networking.k8s.io/v1
metadata:
  name: blue
  labels:
    app: blue
spec:
  parentRefs:
  - name: nginx-gateway
    namespace: nginx-gateway
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /blue
    backendRefs:
    - kind: Service
      name: blue
      port: 80
EOF

kubectl apply -f - <<EOF
kind: HTTPRoute
apiVersion: gateway.networking.k8s.io/v1
metadata:
  name: green
  labels:
    app: green
spec:
  parentRefs:
  - name: nginx-gateway
    namespace: nginx-gateway
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /green
    backendRefs:
    - kind: Service
      name: green
      port: 80
EOF

curl --connect-to ::$WORKER_IP:30080 example.com/blue
curl --connect-to ::$WORKER_IP:30080 example.com/green
```

### Weight

```bash
kubectl apply -f - <<EOF
kind: HTTPRoute
apiVersion: gateway.networking.k8s.io/v1
metadata:
  name: weight
spec:
  parentRefs:
  - name: nginx-gateway
    namespace: nginx-gateway
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /weight
    backendRefs:
    - kind: Service
      name: blue
      port: 80
      weight: 50
    - kind: Service
      name: green
      port: 80
      weight: 50
EOF

curl --connect-to ::$WORKER_IP:30080 example.com/weight
curl --connect-to ::$WORKER_IP:30080 example.com/weight
curl --connect-to ::$WORKER_IP:30080 example.com/weight
curl --connect-to ::$WORKER_IP:30080 example.com/weight
```

### Review

- Create two deployments `blue` and `green` with image `mendhak/http-https-echo:31`.
- The deployment should be accessible on `loadbalance.com` with 80:20 balancing between blue and green. Use HTTPRoute with name `loadbalance` to achieve this.
- Run `curl -s --connect-to ::$WORKER_IP:30180 loadbalance.com | grep hostname` multiple times to check the result.

## Network Policy

Limit network connection between pods.

### Deny from other namespace

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
kubectl run test -it -n dev --rm --image=kubenesia/kubebox -- sh
wget -qO- --timeout=2 kubeapp.default # fail
kubectl run test -it --rm --image=kubenesia/kubebox -- sh
wget -qO- --timeout=2 kubeapp # ok

# Remove NetworkPolicy
kubectl delete netpol deny-from-other-namespaces
```

### Allow only from pod with specific labels
```bash
kubectl create ns marketplace

kubectl create deployment backend --namespace=marketplace --image=nginx:1.27.2
kubectl create deployment frontend --namespace=marketplace --image=nginx:1.27.2
kubectl create deployment experiment --namespace=marketplace --image=nginx:1.27.2

kubectl expose deployment backend --port=80
kubectl expose deployment frontend --port=80
kubectl expose deployment experiment --port=80

cat <<EOF >marketplace-netpol.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-backend-from-frontend
  namespace: marketplace
spec:
  podSelector:
    matchLabels:
      app: backend
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontendn
EOF
kubectl apply -f marketplace-netpol.yaml

kubectl exec -n marketplace -ti deployment/frontend -- bash
curl -m 5 -s backend.marketplace # ok

kubectl exec -n marketplace -ti deployment/experiment -- bash
curl -m 5 -s backend.marketplace # fail
```

### Allow only from specific namespace
```bash
kubectl create ns rnd
kubectl create ns finops
kubectl label ns rnd tier=public
kubectl label ns finops tier=private

cat <<EOF >allow-backend-from-namespaces.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-backend-from-namespaces
  namespace: marketplace
spec:
  podSelector:
    matchLabels:
      app: backend
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          tier: private
EOF

kubectl apply -f allow-backend-from-namespaces.yaml
```

## Review

- Create new namespaces `ride` and `food`.
- Inside namespace `ride`, create 3 deployments:
  - `gateway`
  - `travel`
  - `shadow`
- Inside namespace `food`, create 2 deployments:
  - `order`
  - `tenant`
  - `promo`
- Create NetworkPolicy `allow-from-ride` on namespace `food` to allow access from namespace `ride`.
- Create NetworkPolicy `allow-travel-from-gateway` on namespace `ride` to only allow pod with label `app=travel` to access pod with label `app=gateway` inside the same namespace.
- All deployments should use image `nginx:1.27.2`.
- All deployments should have a Service with type `ClusterIP`.
