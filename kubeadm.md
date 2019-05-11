# How to run Kubernetes (k8s) at home
 If you want to run a full on kubernetes cluster at home (so you can do real testing, like draining nodes, etc) you're going to need more than minikube. 
 
 The good news is that it's really easy as long as you can spin up 3 ubuntu 16.04 vm's. I run kvm/libvirt and manage my vm's with virt-manager. Setup of that is beyond the scope of this document (see [Ubuntu Documentation](https://help.ubuntu.com/community/KVM/Installation) )
 
## Prerequiretes
- 3 Ubuntu16.04 vm's.
- Docker
- Kubeadm
- Private Docker Registry (optional)
- 32GB of disk space per node
  
## Install the  Prerequisites

Follow [this](https://kubernetes.io/docs/setup/independent/install-kubeadm/#installing-kubeadm-kubelet-and-kubectl) guide to install the prerequisites
```https://kubernetes.io/docs/setup/independent/install-kubeadm/#installing-kubeadm-kubelet-and-kubectl```

## Setup a private Docker registry (optional)

If you will be using this cluster to spin up private image builds, you may need to be able to pull from a private registry.

_sourced from https://docs.docker.com/registry/deploying/_

You'll need to run this registry from somewhere *other* than one of the VM's you'll be using for your K8's cluster.

Let's assume you have a docker server running at `10.0.0.3`

1. Ensure docker is running on your docker server
2. On your docker server (_10.0.0.3 in our example_), start the registry

   ```sh
   docker run -d -p 5000:5000 --restart=always --name registry registry:2
   ```
  
3. On each k8s node update docker settings

   ```sh
    cat > /etc/docker/daemon.json <<EOF
    {
      "exec-opts": ["native.cgroupdriver=systemd"],
      "log-driver": "json-file",
      "log-opts": {
        "max-size": "100m"
      },
      "storage-driver": "overlay2",
      "insecure-registries":["10.0.0.3:5000"]
    }
    EOF
   ```


## Setup your cluster
On the master:

`kubeadm init  --pod-network-cidr=192.168.0.0/16`

`export KUBECONFIG=/etc/kubernetes/admin.conf`

 **Save output** *it should look something like below*: 
 `kubeadm join 10.0.0.168:6443 --token s29wj8.0ejnla7c46ikb41d --discovery-token-ca-cert-hash sha256:08860729203944ede7718b509bafa3ff8a0941d10b4c8f734ebfe74963713172`

#### Enable Calico Networking
1. Deploy Calico Pod
`kubectl apply -f https://docs.projectcalico.org/v3.1/getting-started/kubernetes/installation/hosted/kubeadm/1.7/calico.yaml`
2. Wait for pods to become ready
`watch kubectl get pods --all-namespaces`
3. Allow pods on master
`kubectl taint nodes --all node-role.kubernetes.io/master-`
4. Join worker nodes with the output from `kubeadm init` above.
5. Verify that all nodes are ready
`kubectl get nodes -o wide`

#### Add other nodes to cluser
`kubeadm join 10.0.0.168:6443 --token s29wj8.0ejnla7c46ikb41d --discovery-token-ca-cert-hash sha256:08860729203944ede7718b509bafa3ff8a0941d10b4c8f734ebfe74963713172`

#### Setup local storagre for Persistent Volume Claims
##### Persistent Volume
```sh
echo 'kind: PersistentVolume
apiVersion: v1
metadata:
  name: local-pv-volume
  labels:
    type: local
spec:
  storageClassName: local-storage 
  capacity:
    storage: 2Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: "/data/"
' > pv.yaml && kubectl create -f pv.yaml
```
##### Persistent Volume Claim
```sh
echo 'kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: local-pv-claim
spec:
  storageClassName: local-storage 
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
' > pvc.yaml && kubectl create -f pvc.yaml
```
Now you can use something like this in your deployments for persistent storage:
```sh
...
      volumes:
      - name: data
        persistentVolumeClaim:
          claimName: local-pv-claim

...
```

#### Add routes for cluster-ip-range
On the master, in `/etc/kubernetes/manifests/kube-apiserver.yaml` you will find `--service-cluster-ip-range=.....` 
Add this subnet to your routes so you can access services in your cluster.

#### Create your first deployment
`kubectl run hello-world --replicas=2 --labels="run=load-balancer-example" --image=gcr.io/google-samples/node-hello:1.0 --port=8080`

#### Expose your service
`kubectl expose deployment hello-world –type=NodePort –name=example-service`

## Your client

Copy `/etc/kubernetes/admin.conf` on your kube master to your local client, and point your env to it

```sh
export KUBECONFIG=./admin.conf

kubectl get pods
```

## Run an ubuntu pod on your new cluster
```sh
kubectl run ubuntu --rm -i --tty --image ubuntu -- bash
```

## k8s Dasboard

From (here)[https://github.com/kubernetes/dashboard]
`https://github.com/kubernetes/dashboard`

1. Install the pod

```sh
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v1.10.1/src/deploy/recommended/kubernetes-dashboard.yaml
```

2. Grant permission to the dashboard
```sh
echo "apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: kubernetes-dashboard
  labels:
    k8s-app: kubernetes-dashboard
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: kubernetes-dashboard
  namespace: kube-system" > perms.yaml && kubectl create -f perms.yaml
```

3. Get a token to access the dashboard
```sh
kc get secret -o json $(kc get secret | egrep 'default-token-.*' | awk '{ print $1 }')  | jq -r '.data.token'
```

4. Conenct to dashboard
`kubectl proxy`
`http://localhost:8001/api/v1/namespaces/kube-system/services/https:kubernetes-dashboard:/proxy/`

## Helm

If you want to get helm running, install the binary
[Install Helm](https://github.com/helm/helm/releases)
```https://github.com/helm/helm/releases```

Setup helm on your cluster
```sh
export KUBECONFIG=./admin.conf
helm init

kubectl create serviceaccount --namespace kube-system tiller
kubectl create clusterrolebinding tiller-cluster-rule --clusterrole=cluster-admin --serviceaccount=kube-system:tiller
kubectl patch deploy --namespace kube-system tiller-deploy -p '{"spec":{"template":{"spec":{"serviceAccount":"tiller"}}}}'
```
## In case you need to start over

#### Clear resource off the cluster
`kubectl delete po,svc,deployments,cronjob,job --al`

#### Reinstall the cluster
Do this on all the nodes, then the master
`kubeadm reset`
