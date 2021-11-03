# Installare un cluster kubernetes su Ubuntu usando k3s + k3d
### Introduzione
Sappiamo che k3s e' una distribuzione leggera di k8s fornita da Rancher.
Si puo' agilmente installare su un cluster di macchine, installando prima sulla macchina master e poi sulle macchine che saranno associate come worker, tramite un token di associazione dopo aver definito un IP statico per le singole macchine.
In questo esempio useremo una sola macchina virtuale Ubuntu su Katacoda che virtualizzera' il cluster tramite Docker: ogni macchina del cluster sara' avviata in un Docker container!

La cosa sembra bizzarra perche' normalmente un cluster di macchine Kubernetes e' composto da macchine e su ogni macchina e' avviato Docker ed ogni pod che il nodo avvia e' un container Docker.
In questo caso, i nodi stessi sono dei container Docker e in essi e' reinstallato virtualmente Docker... o per meglio dire, una parte soltanto dello stac Docker, ovvero crictl che avvia nel container nodo, altri container pod.

Quando facciamo:
```kubectl exec -it <pod-id> bash ```
stiamo entrando nel container del pod tramite kubectl che contatta le API di kubernetes dal master. Ma in realta' e' come se facessimo un 
``` ssh root@<node-ip-address>```
e poi un
```docker exec -it <docker-pod-id> bash```

Se usiamo k3d per virtualizzare un intero cluster su una sola macchina, allora sicuramente la prima opzione e' ancora valida:
```kubectl exec -it <pod-id> bash ```
ma per la seconda invece di entrare nel container nodo tramite ssh entreremo tramite docker:
```docker exec -it <node-container-id> bash``` entriamo nel nodo container
```crictl ps``` lista dei container attivi nel container nodo
```crictl exec -it <pod-id> bash``` interagiamo con il pod
Oppure in una sola istruzione:
```docker exec -it <node-container-id> crictl images```
```docker exec -it <node-container-id> crictl ps```
```docker exec -it <node-container-id> crictl exec -it <pod-id> bash```

ad esempio:
```docker exec -it k3d-demo-agent-0 crictl exec -it 39ab217962e1c bash```

### Esempio pratico

Andare su [https://www.katacoda.com/courses/ubuntu/playground]

Installare k3d e kubectl
```
$ curl -s https://raw.githubusercontent.com/rancher/k3d/main/install.sh | bash
Preparing to install k3d into /usr/local/bin
k3d installed into /usr/local/bin/k3d
Run 'k3d --help' to see what you can do with it.
$ snap install kubectl --classic
2020-12-04T17:33:01Z INFO Waiting for automatic snapd restart...
kubectl 1.19.4 from Canonical✓ installed
```
Creare un cluster con k3d
```
$ k3d cluster create demo --servers 1 --agents 2
```
Ottenere con kubectl le info del cluster:
```
$ kubectl cluster-info
Kubernetes master is running at https://0.0.0.0:46285
CoreDNS is running at https://0.0.0.0:46285/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy
Metrics-server is running at https://0.0.0.0:46285/api/v1/namespaces/kube-system/services/https:metrics-server:/proxy
```
Conoscere i nodi (macchine virtuali) del cluster:
```
$ kubectl get nodes
NAME                STATUS   ROLES    AGE   VERSION
k3d-demo-agent-1    Ready    <none>   21s   v1.19.4+k3s1
k3d-demo-server-0   Ready    master   28s   v1.19.4+k3s1
k3d-demo-agent-0    Ready    <none>   22s   v1.19.4+k3s1
```
Creiamo un Deployment:
```
$ cat > nginx-deployment.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
  labels:
    app: nginx
  annotations:
    monitoring: "true"
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - image: nginx
        name: nginx
        ports:
        - containerPort: 80
        resources:
          limits:
            memory: "2Gi"
            cpu: "1000m"
          requests:
            memory: "1Gi"
            cpu: "500m"
---
apiVersion: v1
kind: Service
metadata:
  labels:
    app: nginx
  name: nginx
  namespace: default
spec:
  type: NodePort
  ports:
  - nodePort: 30500
    port: 80
    protocol: TCP
    targetPort: 80
  selector:
    app: nginx
EOF
```
Deployamo nel cluster:
```
$ kubectl apply -f nginx-deployment.yaml
deployment.apps/nginx created
service/nginx created
```
Vediamo un po' cosa c'e' nel cluster:
```
$ kubectl get pods
NAME                    READY   STATUS    RESTARTS   AGE
nginx-55b55d957-9xf5k   1/1     Running   0          2m58s

$ kubectl get deployments
NAME    READY   UP-TO-DATE   AVAILABLE   AGE
nginx   1/1     1            1           3m4s

$ kubectl get services
NAME         TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
kubernetes   ClusterIP   10.43.0.1       <none>        443/TCP        7m47s
nginx        NodePort    10.43.119.197   <none>        80:30500/TCP   21s

$ kubectl get namespaces
NAME              STATUS   AGE
default           Active   8m3s
kube-system       Active   8m3s
kube-public       Active   8m3s
kube-node-lease   Active   8m3s

$ kubectl get nodes
NAME                STATUS   ROLES    AGE     VERSION
k3d-demo-agent-0    Ready    <none>   9m54s   v1.19.4+k3s1
k3d-demo-agent-1    Ready    <none>   9m53s   v1.19.4+k3s1
k3d-demo-server-0   Ready    master   10m     v1.19.4+k3s1

$ kubectl describe node k3d-demo-agent-0 | grep IP
  InternalIP:  172.19.0.3

$ kubectl describe node k3d-demo-agent-1 | grep IP
  InternalIP:  172.19.0.4

$ curl http://172.19.0.3:30500
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
...
</html>

$ curl http://172.19.0.4:30500
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
...
</html>
```
Entrambi gli IP dei nodi con la porta NodePort aperta per accedervi hanno restituito la Welcome Page di Nginx.
Ma in realta' solo in uno dei nodi e' stato deployato Nginx. Questo dimostra che il cluster fa da proxy tramite il suo sistema interno di iptables.

Vediamo i pod deployati dal sistema nel nodo k3d-demo-agent-0:
```
$ kubectl get pods --all-namespaces -o wide --field-selector spec.nodeName=k3d-demo-agent-0
NAMESPACE     NAME                                     READY   STATUS    RESTARTS   AGE     IP          NODE               NOMINATED NODE   READINESS GATES
kube-system   local-path-provisioner-7ff9579c6-qkl6g   1/1     Running   0          11m     10.42.2.2   k3d-demo-agent-0   <none>   <none>
kube-system   svclb-traefik-hpxzb                      2/2     Running   0          11m     10.42.2.3   k3d-demo-agent-0   <none>   <none>
default       nginx-55b55d957-9xf5k                    1/1     Running   0          7m25s   10.42.2.4   k3d-demo-agent-0   <none>   <none>
```
Vediamo i pod deployati dal sistema nel nodo k3d-demo-agent-1:
```
$ kubectl get pods --all-namespaces -o wide --field-selector spec.nodeName=k3d-demo-agent-1
NAMESPACE     NAME                             READY   STATUS    RESTARTS   AGE   IP          NODE               NOMINATED NODE   READINESS GATES
kube-system   coredns-66c464876b-fdl4t         1/1     Running   0          11m   10.42.0.3   k3d-demo-agent-1   <none>           <none>
kube-system   metrics-server-7b4f8b595-lz8nm   1/1     Running   0          11m   10.42.0.2   k3d-demo-agent-1   <none>           <none>
kube-system   svclb-traefik-tq62l              2/2     Running   0          11m   10.42.0.4   k3d-demo-agent-1   <none>           <none>
```
Come si nota il pod di Nginx e' stato deployato solo nel primo nodo.

Come abbiamo detto prima, in k3d i nodi non sono macchine fisiche ma container docker avviati nella nostra macchina Ubuntu. Infatti ispezionando i container attivi nella macchina:
```
$ docker ps
CONTAINER ID        IMAGE                      COMMAND                  CREATED             STATUS              PORTS         NAMES
6b23ca8619cf        rancher/k3d-proxy:v3.4.0   "/bin/sh -c nginx-pr…"   12 minutes ago      Up 12 minutes       80/tcp, 0.0.0.0:46285->6443/tcp   k3d-demo-serverlb
48454de0c41b        rancher/k3s:v1.19.4-k3s1   "/bin/k3s agent"         12 minutes ago      Up 12 minutes         k3d-demo-agent-1
3e6492c44d78        rancher/k3s:v1.19.4-k3s1   "/bin/k3s agent"         12 minutes ago      Up 12 minutes         k3d-demo-agent-0
b620dd19e494        rancher/k3s:v1.19.4-k3s1   "/bin/k3s server --t…"   12 minutes ago      Up 12 minutes         k3d-demo-server-0
```
Vediamo come docker abbia un container per ogni nodo
Ogni nodo (macchina) dovrebbe avere docker installato all'interno. In questo caso k3s usa solo una parte dello stack docker ovvero crictl quindi non abbiamo dei container docker con all'interno docker e dei container docker ma dei container docker (nodi) con all'interno dei container (e delle immagini) crictl.
Ispezionando il primo nodo alla ricerca delle immagini:
```
$ docker exec -it k3d-demo-agent-0 crictl images
IMAGE                                      TAG                 IMAGE ID            SIZE
docker.io/library/nginx                    latest              bc9a0695f5712       53.6MB
docker.io/rancher/klipper-lb               v0.1.2              897ce3c5fc8ff       2.71MB
docker.io/rancher/local-path-provisioner   v0.0.14             e422121c9c5f9       13.4MB
docker.io/rancher/pause                    3.1                 da86e6ba6ca19       327kB
```
Ispezionando il secondo nodo:
```
$ docker exec -it k3d-demo-agent-1 crictl images
IMAGE                               TAG                 IMAGE ID            SIZE
docker.io/rancher/coredns-coredns   1.6.9               4e797b3234604       13.4MB
docker.io/rancher/klipper-lb        v0.1.2              897ce3c5fc8ff       2.71MB
docker.io/rancher/metrics-server    v0.3.6              9dd718864ce61       10.5MB
docker.io/rancher/pause             3.1                 da86e6ba6ca19       327kB
```
Vedendo i container attivi nei nodi-container:
```
$ docker exec -it k3d-demo-agent-1 crictl ps
CONTAINER           IMAGE               CREATED             STATE               NAME                ATTEMPT             POD ID
de2d4ec199efa       897ce3c5fc8ff       13 minutes ago      Running             lb-port-443         0                   5156b5b979904
e8defa292d866       897ce3c5fc8ff       13 minutes ago      Running             lb-port-80          0                   5156b5b979904
f1394af462347       9dd718864ce61       13 minutes ago      Running             metrics-server      0                   a159009bba8df
8955ab9352aa2       4e797b3234604       13 minutes ago      Running             coredns             0                   342790501f649


$ docker exec -it k3d-demo-agent-0 crictl ps
CONTAINER           IMAGE               CREATED             STATE               NAME                     ATTEMPT             POD ID
39ab217962e1c       bc9a0695f5712       9 minutes ago       Running             nginx                    0                   c59c877dda622
5f30bd08bfdcf       897ce3c5fc8ff       13 minutes ago      Running             lb-port-443              0                   9232e854f4d8e
b899e526f6895       897ce3c5fc8ff       13 minutes ago      Running             lb-port-80               0                   9232e854f4d8e
049047792f684       e422121c9c5f9       13 minutes ago      Running             local-path-provisioner   0                   171d7400e9954
```

Entriamo nel primo nodo e dal primo nodo vediamo i container attivi con critcl. Se proviamo con docker, questo comando all'interno del container nodo non viene trovato.
Entriamo con crictl nel container interno di nginx e creiamo un file example.txt
```
$ docker exec -it k3d-demo-agent-0 sh
/ # ls
bin  dev  etc  k3d  lib  proc  run  sbin  sys  tmp  usr  var
/ # crictl ps
CONTAINER           IMAGE               CREATED             STATE               NAME                     ATTEMPT             POD ID
39ab217962e1c       bc9a0695f5712       10 minutes ago      Running             nginx                    0                   c59c877dda622
5f30bd08bfdcf       897ce3c5fc8ff       14 minutes ago      Running             lb-port-443              0                   9232e854f4d8e
b899e526f6895       897ce3c5fc8ff       14 minutes ago      Running             lb-port-80               0                   9232e854f4d8e
049047792f684       e422121c9c5f9       14 minutes ago      Running             local-path-provisioner   0                   171d7400e9954

/ # docker ps
sh: docker: not found

/ # crictl exec -it 39ab217962e1c bash
root@nginx-55b55d957-9xf5k:/# touch example.txt
root@nginx-55b55d957-9xf5k:/# ls
bin   dev                  docker-entrypoint.sh  example.txt  lib    media  opt   root  sbin  sys  usr
boot  docker-entrypoint.d  etc                   home         lib64  mnt    proc  run   srv   tmp  var
root@nginx-55b55d957-9xf5k:/# exit
exit
/ # exit
```
Se proviamo ad entrare nel primo nodo e nel container nginx tramite kubectl abbiamo accesso molto piu' velocemente e troviamo il file creato prima:
```
$ kubectl get pods
NAME                    READY   STATUS    RESTARTS   AGE
nginx-55b55d957-9xf5k   1/1     Running   0          12m

$ kubectl exec -it nginx-55b55d957-9xf5k bash
root@nginx-55b55d957-9xf5k:/# ls | grep example.txt
example.txt
root@nginx-55b55d957-9xf5k:/# exit
exit
```
