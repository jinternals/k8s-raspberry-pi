# Setup k8s cluster - raspberry-pi 

![Cluster Image](docs/cluster.jpg)

Hardware:
  - [6 x raspberry pic 4B 8GB](https://www.raspberrypi.com/products/raspberry-pi-4-model-b/)
  - [1 x 8-Port Gigabit Easy Smart Switch ](https://www.tp-link.com/in/business-networking/easy-smart-switch/tl-sg108e/v6/)
  - [1 x AC750 Wireless Travel Router](https://www.tp-link.com/in/home-networking/wifi-router/tl-wr902ac/)
  - [7 x 1FT CAT 6 Patch C Cables](https://www.amazon.in/gp/product/B005RCG0FK/ref=ppx_yo_dt_b_asin_title_o01_s00?ie=UTF8&psc=1)
  - [1 x Anker Power Port 6 (60W)](https://www.crazypi.com/raspberry-pi-cluster-power-supply-60w)
  - [Raspberry Pi Cluster 6 Layer Rack](https://www.crazypi.com/raspberry-pi-cluster-6-layer?search=cluster&description=true)
  - [Hoteon Power Strip plus + Cable Management Box](https://www.amazon.in/gp/product/B094NDJGYL/ref=ppx_yo_dt_b_asin_title_o03_s00?ie=UTF8&psc=1)
  
### Install k3s
```shell
ansible-playbook -i hosts playbook/k3s/k3s.yml
```

### 1. Install metallb

```shell
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.12.1/manifests/namespace.yaml
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.12.1/manifests/metallb.yaml
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

=======
### SSL certs setup 
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: cloudflare-api-token-secret
  namespace: cert-manager
type: Opaque
data:
  api-token: <token>
---
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    email: <your-email>
    server: https://acme-v02.api.letsencrypt.org/directory
    privateKeySecretRef:
      name: letsencrypt-prod
    solvers:
      - dns01:
          cloudflare:
            email: <your-email>
            apiKeySecretRef:
              name: cloudflare-api-token-secret
              key: api-token
---
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: <your-email>
spec:
  acme:
    email: pandeymradul@gmail.com
    server: https://acme-staging-v02.api.letsencrypt.org/directory
    privateKeySecretRef:
      name: letsencrypt-stating
    solvers:
      - dns01:
          cloudflare:
            email: <your-email>
            apiKeySecretRef:
              name: cloudflare-api-token-secret
              key: api-token
```

```shell
ansible-playbook -i hosts playbook/longhorn/longhorn.yaml

kubectl apply -f https://raw.githubusercontent.com/longhorn/longhorn/master/deploy/longhorn.yaml
```
### Blogs:

  - https://banzaicloud.com/blog/load-balancing-on-prem/
  - https://fredrickb.com/2021/08/22/recreating-the-raspberry-pi-homelab-with-kubernetes/
  - https://tothecloud.dev/using-certmanager-with-cloudflare-and-kubernetes/
