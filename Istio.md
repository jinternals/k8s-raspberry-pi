# Steps for Enabling Istio - Gateway API support


### 1. Install the Gateway API CRDs and Install Istio using the minimal profile
```shell
kubectl get crd gateways.gateway.networking.k8s.io &> /dev/null || \
  { kubectl kustomize "github.com/kubernetes-sigs/gateway-api/config/crd?ref=444631bfe06f3bcca5d0eadf1857eac1d369421d" | kubectl apply -f -; }

istioctl install --set profile=minimal -y
```
### 2. Install Istio addons

```shell
kubectl apply -f ~/istio-1.21.0/samples/addons/ 
```

### 3. Install cert-manager
```shell
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.14.4/cert-manager.yaml
```
### 4. Enable ExperimentalGatewayAPISupport in  cert-manager-controller:

![enable-cert-manager-flag.png](images%2Fenable-cert-manager-flag.png)

### 5. Setup SSL cert issuer(below example is using cloudflare)
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
    email: <your-email>
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

#### 6. Deploy Sample Application

```shell
apiVersion: v1
kind: Namespace
metadata:
  name: istio-demo
  labels:
    istio-injection: enabled
---
apiVersion: v1
kind: Service
metadata:
  name: demo-svc
  namespace: istio-demo
spec:
  type: ClusterIP
  selector:
    app: demo-app
  ports:
    - port: 80
      targetPort: 80
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: demo-deployment
  namespace: istio-demo
  labels:
    app: demo-app
    version: v1
spec:
  replicas: 1
  selector:
    matchLabels:
      app: demo-app
      version: v1
  template:
    metadata:
      labels:
        app: demo-app
        version: v1
    spec:
      containers:
        - name: demo-app
          image: ealen/echo-server:latest
          ports:
            - containerPort: 80
---
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: k8s-gateway
  namespace: istio-demo
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-stating
spec:
  gatewayClassName: istio
  listeners:
    - name: http-listener
      hostname: "*.homelabs.co.in"
      port: 80
      protocol: HTTP
      allowedRoutes:
        namespaces:
          from: All
    - name: https-listener
      hostname: "test.homelabs.co.in"
      port: 443
      protocol: HTTPS
      tls:
        mode: Terminate
        certificateRefs:
          - name: "test-homelabs-cert"
      allowedRoutes:
        namespaces:
          from: All
---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: insecure-http-route
  namespace: istio-demo
spec:
  parentRefs:
    - kind: Gateway
      name: k8s-gateway
      sectionName: http-listener
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /
      filters:
        - type: RequestHeaderModifier
          requestHeaderModifier:
            add:
              - name: HTTPRoute
                value: insecure-http-route
      backendRefs:
        - name: demo-svc
          port: 80
---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: secure-http-route
  namespace: istio-demo
spec:
  parentRefs:
    - kind: Gateway
      name: k8s-gateway
      sectionName: https-listener
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /
      filters:
        - type: RequestHeaderModifier
          requestHeaderModifier:
            add:
              - name: HTTPRoute
                value: secure-http-route
      backendRefs:
        - name: demo-svc
          port: 80
```

### 7. Demo Output
![istio-demo.png](images%2Fistio-demo.png)
