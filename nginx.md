# Steps for Enabling Nginx - Gateway API support

### 1. Install Gateway API CRDs (v1.3.0)
```shell
  kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.3.0/standard-install.yaml
```
### 2. Install cert-manager (v1.18.2)
```shell
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.18.2/cert-manager.yaml
```

### 3. Installs the NGINX-specific GatewayClass & other CRDs.
```shell
kubectl delete -f https://raw.githubusercontent.com/nginx/nginx-gateway-fabric/v1.6.2/deploy/crds.yaml
```

### 4. Deploys the NGINX Gateway Fabric controller.
```shell
kubectl apply -f https://raw.githubusercontent.com/nginx/nginx-gateway-fabric/v1.6.2/deploy/default/deploy.yaml
```

### 5. Demo Application
> ⚠️ **Warning**:  The annotation cert-manager.io/cluster-issuer: letsencrypt-prod does not make sense or have any effect in the context of NGINX Gateway Fabric.
> That annotation is only meaningful to Istio's implementation of Gateway API, which has custom logic to:
> Watch for Gateway resources with that annotation
> Automatically create Certificate resources for TLS

```yaml
annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
```

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: nginx-demo
---
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: secure-homelabs-cert
  namespace: nginx-demo
spec:
  secretName: test-homelabs-cert
  dnsNames:
    - secure.homelabs.co.in
  issuerRef:
    name: letsencrypt-prod
    kind: ClusterIssuer
  usages:
    - digital signature
    - key encipherment
---
apiVersion: v1
kind: Service
metadata:
  name: demo-svc
  namespace: nginx-demo
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
  namespace: nginx-demo
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
  namespace: nginx-demo
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
  gatewayClassName: nginx
  listeners:
    - name: http-listener
      hostname: "*.homelabs.co.in"
      port: 80
      protocol: HTTP
      allowedRoutes:
        namespaces:
          from: All
    - name: https-listener
      hostname: "secure.homelabs.co.in"
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
  namespace: nginx-demo
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
  namespace: nginx-demo
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