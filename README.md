# kubernetes-cluster

# How to deploy a Kubernetes cluster
> HERMOSILLA Nicolas
> M2 Infra Cloud Sécurité

# Infrastructure provisionning
> ⚠ Ajouter la clé SSH sur scaleway avant de faire toute manipulation. 
> Cela nous permettra d'ajouter la clé SSH aux instances et de s'y connecter

- Récupérer le terraform permettant de faire le déploiement
git clone git@github.com:Arcahub/kube-install-tuto.git

- Créer une API key pour l'utilisateur en sélectionnant le projet M1-M2 Conteneur et Orchestration. Ajouter ensuite la clé d'accès et la clé secrète à l'aide de la commande suivante:
```
scw init
```
- Modifier les fichiers variables.yaml kube-install-tuto/terraform/modules/k8s/modules:
```
type : DEV1-M
```
- Aller dans le dossier terraform et déployer le projet
```
cd kube-install-tuto/terraform
terraform init
terraform plan
terraform apply -var="project_name=nicolas-kube"
```
- Récupérer les IPs
```
terraform output

control_plane_ip = "10.74.62.113"
public_gateway_ip = "212.47.253.167"
worker_ips = [
  "10.71.96.47",
  "10.70.152.77",
]
```

# Kubernetes installation with kubeadm
```
ssh -J bastion@212.47.253.167:61000 root@10.74.62.113
ssh -J bastion@212.47.253.167:61000 root@10.71.96.47
ssh -J bastion@212.47.253.167:61000 root@10.70.152.77
```
- Préparer les nodes. Lancer les commandes sur les 3 instances
```
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

# sysctl params required by setup, params persist across reboots
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

# Apply sysctl params without reboot
sudo sysctl --system
```
- Installer containerd
```
# Install required packages for https repository
sudo apt-get update && sudo apt-get install -y apt-transport-https ca-certificates curl
# Add Docker’s official GPG key
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
# Add Docker repository
sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
# Update package manager index
sudo apt-get update
```

```
sudo apt-get install -y containerd.io
```

```
sudo mkdir -p /etc/containerd
sudo containerd config default > /etc/containerd/config.toml
```

- Modifier le fichier /etc/containerd/config.toml en mettant la valeur du SystemdCgroup à `true`. Redémarrer ensuite le service containerd
```
vim /etc/containerd/config.toml
```
![](https://i.imgur.com/zdOwHtS.png)

```
sudo systemctl restart containerd
```
- Tester le bon fonctionnement du service containerd
```
ctr images pull docker.io/library/hello-world:latest
sudo ctr run --rm docker.io/library/hello-world:latest hello-world
ctr images rm docker.io/library/hello-world:latest
```

- Installer les paquets kubeadm, kubelet et kubectl
```
# Install required packages for https repository
sudo apt-get update && sudo apt-get install -y apt-transport-https curl

# Add Kubernetes’s official GPG key
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -

# Add Kubernetes repository
cat <<EOF | sudo tee /etc/apt/sources.list.d/kubernetes.list
deb https://apt.kubernetes.io/ kubernetes-xenial main
EOF

# Update package manager index
sudo apt-get update

# Install kubeadm, kubelet and kubectl with the exact same version or else components could be incompatible
sudo apt-get install -y kubelet=1.25.0-00 kubeadm=1.25.0-00 kubectl=1.25.0-00

# Hold the version of the packages
sudo apt-mark hold kubelet kubeadm kubectl
```

# Setting up control plane node
-  Se connecter au node control plane et initier le nœud avec kubeadm
```
ssh -J bastion@212.47.253.167:61000 root@10.74.62.113
kubeadm init
```
![](https://i.imgur.com/kDj5kwk.png)

- Modifier le fichier kubeconfig pour pouvoir utiliser kubectl et utiliser le cluster
```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
- Vérifier les nodes et les pods. Nous remarquons que le node control-plane n'est pas encore prêt
```
kubectl get nodes

NAME                        STATUS     ROLES           AGE     VERSION
nicolas-kube-controlplane   NotReady   control-plane   4m54s   v1.25.0
```
```
kubectl get pods --all-namespaces

NAMESPACE     NAME                                                READY   STATUS    RESTARTS   AGE
kube-system   coredns-565d847f94-8g78h                            0/1     Pending   0          5m17s
kube-system   coredns-565d847f94-dxdqc                            0/1     Pending   0          5m17s
kube-system   etcd-nicolas-kube-controlplane                      1/1     Running   0          5m20s
kube-system   kube-apiserver-nicolas-kube-controlplane            1/1     Running   0          5m20s
kube-system   kube-controller-manager-nicolas-kube-controlplane   1/1     Running   0          5m20s
kube-system   kube-proxy-857mc                                    1/1     Running   0          5m17s
kube-system   kube-scheduler-nicolas-kube-controlplane            1/1     Running   0          5m22s
```
- Installer le plugin CNI (Container Network Interface) pour que les pods puissent se communiquer entre eux
```
kubectl apply -f https://github.com/weaveworks/weave/releases/download/v2.8.1/weave-daemonset-k8s.yaml
```
- Attendre que le pod weave-net soit prêt
```
kubectl -n kube-system wait pod -l name=weave-net --for=condition=Ready --timeout=-1s
pod/weave-net-s6v44 condition met

kubectl get pods -l name=weave-net -n kube-system
NAME              READY   STATUS    RESTARTS      AGE
weave-net-s6v44   2/2     Running   1 (96s ago)   103s
```
- Vérifier que le node control plane est maintenant prêt
```
kubectl get nodes
NAME                        STATUS   ROLES           AGE   VERSION
n-kube-controlplane   Ready    control-plane   10m   v1.25.0
```
```
kubectl get pods --all-namespaces
NAMESPACE     NAME                                                READY   STATUS    RESTARTS        AGE
kube-system   coredns-565d847f94-8g78h                            1/1     Running   0               12m
kube-system   coredns-565d847f94-dxdqc                            1/1     Running   0               12m
kube-system   etcd-nicolas-kube-controlplane                      1/1     Running   0               12m
kube-system   kube-apiserver-nicolas-kube-controlplane            1/1     Running   0               12m
kube-system   kube-controller-manager-nicolas-kube-controlplane   1/1     Running   0               12m
kube-system   kube-proxy-857mc                                    1/1     Running   0               12m
kube-system   kube-scheduler-nicolas-kube-controlplane            1/1     Running   0               12m
kube-system   weave-net-s6v44                                     2/2     Running   1 (4m26s ago)   4m33s
```

# Setting up worker node
- Toujours sur le control plane, créer un token pour relier les 2 workers au cluster
```
kubeadm token create --print-join-command
kubeadm join 192.168.1.55:6443 --token iuqy4s.sqv2yfibfdfw746p --discovery-token-ca-cert-hash sha256:2908618fb5d9039b23230718031e0bc74dfcef35e7739137eb3980b2ef3eefa6
```

Se connecter aux 2 nodes puis lancer la commande générée pour ajouter les ajouter au cluster
```
ssh -J bastion@212.47.253.167:61000 root@10.71.96.47
ssh -J bastion@212.47.253.167:61000 root@10.70.152.77
kubeadm join 192.168.1.55:6443 --token iuqy4s.sqv2yfibfdfw746p --discovery-token-ca-cert-hash sha256:2908618fb5d9039b23230718031e0bc74dfcef35e7739137eb3980b2ef3eefa6
```
![](https://i.imgur.com/yORX3IZ.png)
- Vérifier sur le control plane que les workers fonctionnent correctement et ajouter un label pour chaque worker
```
ssh -J bastion@212.47.253.167:61000 root@10.74.62.113
kubectl get nodes
NAME                        STATUS   ROLES           AGE     VERSION
nicolas-kube-controlplane   Ready    control-plane   18m     v1.25.0
nicolas-kube-node-1         Ready    <none>          2m29s   v1.25.0
nicolas-kube-node-2         Ready    <none>          89s     v1.25.0
```
```
kubectl label node nicolas-kube-node-1 node-role.kubernetes.io/worker=worker
node/nicolas-kube-node-1 labeled

kubectl label node nicolas-kube-node-2 node-role.kubernetes.io/worker=worker
node/nicolas-kube-node-2 labeled
```

# Understanding what we have done
## Les pods statiques
- Verifier sur le control plane que les fichiers dans /etc/kubernetes/manifests correspondent bien aux pods avec comme namespace `kube-system`
```
sudo ls /etc/kubernetes/manifests
kubectl get pods --namespace=kube-system
``` 

- Supprimer le pod kube-apiserver pour voir si le pod se recréer automatiquement. Kubelet vérifiera en permanence le dossier du static pod et recréera le pod s'il est détruit.
```
kubectl delete pod kube-apiserver-nicolas-kube-controlplane --namespace=kube-system
pod "kube-apiserver-nicolas-kube-controlplane" deleted
```
```
kubectl get pods --namespace=kube-system
```

- Simuler la suppression du fichier du pod kube-apiserver. On obtient une erreur car kubectl n'arrive pas à contacter le server API.
```
sudo mv /etc/kubernetes/manifests/kube-apiserver.yaml ~/kube-apiserver.yaml
kubectl get pods --namespace=kube-system
```
- Remettons le fichier dans le bon dossier. Le pod est de nouveau présent.
```
sudo mv ~/kube-apiserver.yaml /etc/kubernetes/manifests/kube-apiserver.yaml
kubectl get pods --namespace=kube-system 
```
- Nous pouvons également ajouter un nouveau fichier pour créer un pod nginx. Le pod nginx sera directement en état running grâce à kubelet. 
```
sudo tee /etc/kubernetes/manifests/nginx.yaml <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: nginx-test
  namespace: default
spec:
  containers:
  - name: nginx
    image: nginx
    ports:
    - containerPort: 80
EOF
```
```
kubectl get pods --namespace=default
```
- Supprimer le fichier nginx pour supprimer le pod et vérifier que le pod soit bien supprimé.
```
sudo rm /etc/kubernetes/manifests/nginx.yaml
kubectl get pods --namespace=default
No resources found in default namespace.
```
## Les certificats
- Vérifier l'expiration des certificats avec `kubeadm`
```
kubeadm certs check-expiration
```
- Il est possible de renouveler les certificats. Il faudra ensuite redémarrer le kubelet.
```
kubeadm certs renew all
sudo systemctl daemon-reload
sudo systemctl restart kubelet
```

# Creating our first deployment
- Déployer une première application nginx avec 3 replicas.
```
kubectl create deployment nginx --image=nginx --replicas=3
kubectl get deployments
NAME    READY   UP-TO-DATE   AVAILABLE   AGE
nginx   3/3     3            3           9s 
```

# Playing with the scheduler
## Manual scheduling
- Déplacer le fichier kube-scheduler.yaml afin de s'assurer qu'aucune modification ne soit effectuée par ce dernier. Vérifier que le pod ne tourne plus.
```
sudo mv /etc/kubernetes/manifests/kube-scheduler.yaml /tmp
kubectl get pods -n kube-system
```
- Créer un pod nginx en créant un fichier nginx.yaml dans manifest pour pouvoir y apporter des modifications.
```
kubectl run nginx --image=nginx --dry-run=client -o yaml > ~/nginx.yaml
```
- Créer le pod nginx
```
kubectl apply -f ~/nginx.yaml
pod/nginx created
```
- Vérifier que le pod s'est bien créé. Le pod est en état pending.
```
kubectl get pods
NAME    READY   STATUS    RESTARTS   AGE  
nginx   0/1     Pending   0          2m29s
```
- Déployer un nouveau pod nginx.
```
vim nginx.yaml
```
![](https://i.imgur.com/rK2CjMx.png)
```
kubectl apply -f ~/nginx.yaml
pod/nginx2 created
```
- Vérifier que le pod est bien dans la liste. Le pod nginx2 est bien en état running.
```
kubectl get pods
```
![](https://i.imgur.com/BqzFv48.png)

- Redéplacer le fichier scheduler et vérifier que le pod nginx fonctionne correctement
```
sudo mv /tmp/kube-scheduler.yaml /etc/kubernetes/manifests
kubectl get pods
```
![](https://i.imgur.com/5cDc8e6.png)

- Supprimer les 2 pods nginx et nginx2
```
kubectl delete pod nginx nginx2
```

## Node selector
- Utiliser un label pour scheduler un pod. Ajouter un label à un node existant
```
kubectl label node nicolas-kube-node-1 node-type=worker
node/nicolas-kube-node-1 labeled
```
- Créer un pod avec le node selector
```
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
  - name: nginx
    image: nginx
  nodeSelector:
    node-type: worker
EOF

pod/nginx created
```
- Vérifier que le pod fonctionne correctement
```
kubectl get pods -o wide
```

- Supprimer le pod
```
kubectl delete pod nginx
pod "nginx" deleted
```

## Taints and tolerations
- Le node control plane a par défaut un taint.
```
kubectl describe node nicolas-kube-controlplane | grep Taints
Taints:             node-role.kubernetes.io/control-plane:NoSchedule
```
- Créer un pod qui ne sera lié à aucun worder mais fonctionnera sur le node control plane
```
cat<<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
    containers:
    - name: nginx
      image: nginx
    tolerations:
    - key: node-role.kubernetes.io/master
      operator: Equal
      value: ""
    - key: node-role.kubernetes.io/control-plane
      operator: Equal
      value: ""
    nodeSelector:
      node-role.kubernetes.io/control-plane: ""
EOF

pod/nginx created
```
- Vérifier que le pod fonctionne correctement
```
kubectl get pods
NAME    READY   STATUS    RESTARTS   AGE
nginx   1/1     Running   0          36s
```
- Supprimer le pod
```
kubectl delete pod nginx
pod "nginx" deleted
```
- Recréer le pod sur le control plane et vérifier son comportement. Le pod se créer bien mais reste en état pending car il n'y a pas de tolérance.
```
cat<<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
    containers:
    - name: nginx
      image: nginx
    nodeSelector:
      node-role.kubernetes.io/control-plane: ""
EOF

pod/nginx created
```
```
kubectl get pods
NAME    READY   STATUS    RESTARTS   AGE
nginx   0/1     Pending   0          48s
```
- Supprimer le pod
```
kubectl delete pod nginx
pod "nginx" deleted
```

# Upgrading cluster version
## Upgrade control plane

- Upgrade kubeadm
```
sudo apt-mark unhold kubeadm && \
sudo apt update && apt install -y kubeadm=1.26.0-00 && \
sudo apt-mark hold kubeadm
```
- Verifier que le upgrade est disponible
```
kubeadm upgrade plan
```
- Upgrader le cluster avec la bonne version
```
kubeadm upgrade apply v1.26.0
```
![](https://i.imgur.com/Rxepyta.png)

- Drainer le node control plane et upgrader kubelet
```
kubectl drain nicolas-kube-controlplane --ignore-daemonsets

node/nicolas-kube-controlplane cordoned
Warning: ignoring DaemonSet-managed Pods: kube-system/kube-proxy-65tmx, kube-system/weave-net-s6v44

sudo apt-mark unhold kubelet && \
sudo apt update && apt install -y kubelet=1.26.0-00 && \
sudo apt-mark hold kubelet
```
- Redémarrer le service kubelet
```
sudo systemctl daemon-reload
sudo systemctl restart kubelet
```
- Déconnecter le node control plane
```
kubectl uncordon nicolas-kube-controlplane
node/nicolas-kube-controlplane uncordoned
```

## Upgrade worker nodes
- Se connecter aux workers
```
ssh -J bastion@212.47.253.167:61000 root@10.71.96.47
ssh -J bastion@212.47.253.167:61000 root@10.70.152.77
```
- Upgrade kubeadm sur chaque worker
```
sudo apt-mark unhold kubeadm && \
sudo apt update && apt install -y kubeadm=1.26.0-00 && \
sudo apt-mark hold kubeadm
```
- Upgrader le node sur chaque worker
```
kubeadm upgrade node
```
![](https://i.imgur.com/1seEQFI.png)

- Drainer les workers sur le control plane puis upgrade kubelet sur chaque worker
```
kubectl drain nicolas-kube-node-1 --ignore-daemonsets
kubectl drain nicolas-kube-node-2 --ignore-daemonsets
```
```
sudo apt-mark unhold kubelet && \
sudo apt update && apt install -y kubelet=1.26.0-00 && \
sudo apt-mark hold kubelet
```
- Redémarrer kubelet sur chaque worker
```
sudo systemctl daemon-reload
sudo systemctl restart kubelet
```
- Déconnecter les workers depuis le control plane
```
kubectl uncordon nicolas-kube-node-1
node/nicolas-kube-node-1 uncordoned

kubectl uncordon nicolas-kube-node-2
node/nicolas-kube-node-2 uncordoned
```
- Vérifier que le cluster fonctionne et que les workers ont la bonne version
```
kubectl get nodes
```

```
kubectl get nodes
NAME                          STATUS   ROLES           AGE    VERSION
nicolas-kube-controlplane     Ready    control-plane   156m   v1.26.0
nicolas-kube-node-1           Ready    worker          137m   v1.26.0
nicolas-kube-node-2           Ready    worker          136m   v1.26.0
```
