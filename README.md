# Setup k8s cluster - raspberry-pi 

![Cluster Image](docs/cluster.jpg)

Hardware:
  - [6 x raspberry pic 4B 8GB](https://www.raspberrypi.com/products/raspberry-pi-4-model-b/)
  - [1 x 8-Port Gigabit Easy Smart Switch ](https://www.tp-link.com/in/business-networking/easy-smart-switch/tl-sg108e/v6/)
  - [1 x AC750 Wireless Travel Router](https://www.tp-link.com/in/home-networking/wifi-router/tl-wr902ac/)
  - [7 x 1FT CAT 6 Patch C Cables](https://www.amazon.in/gp/product/B005RCG0FK/ref=ppx_yo_dt_b_asin_title_o01_s00?ie=UTF8&psc=1)
  - [1 x Anker Power Port 6 (60W)](https://www.crazypi.com/raspberry-pi-cluster-power-supply-60w)
  - [Hoteon Power Strip plus + Cable Management Box](https://www.amazon.in/gp/product/B094NDJGYL/ref=ppx_yo_dt_b_asin_title_o03_s00?ie=UTF8&psc=1)
  
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


### Blogs:

  - https://banzaicloud.com/blog/load-balancing-on-prem/
  - https://fredrickb.com/2021/08/22/recreating-the-raspberry-pi-homelab-with-kubernetes/
