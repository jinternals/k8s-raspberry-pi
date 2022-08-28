# Setup k8s cluster - raspberry-pi 

![Cluster Image](docs/cluster.jpg)

### Install k3s
```shell
ansible-playbook -i hosts playbook/k3s/k3s.yml
```

### 1. Install metallb

```shell
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.11.0/manifests/namespace.yaml
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.11.0/manifests/metallb.yaml
kubectl apply -f apps/metallb/config.yaml
```

### 2. install Ingress-nginx

```shell
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/static/provider/baremetal/deploy.yaml

```

```shell
edit ingress-nginx-controller 
replace NodePort to LoadBalancer
```

### 3. Install cert-manager
```shell
kubectl apply -f apps/ingress_class.yaml
helm repo add jetstack https://charts.jetstack.io
```

```shell
helm install \
  cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --version v1.9.1 \
  --set installCRDs=true
```


### 4. Install rancher

```shell
helm repo add rancher-latest https://releases.rancher.com/server-charts/latest
```

```shell
kubectl create namespace cattle-system

helm install rancher rancher-latest/rancher \
    --namespace cattle-system \
    --set hostname=rancher.homelab.com \
    --set bootstrapPassword=admin
    
    
kubectl expose deployment rancher -n cattle-system --type=LoadBalancer --name=rancher-lb --port=443
    
```


```shell

```
