## NodePort Service:

In a Kubernetes cluster, iptables plays a crucial role in managing network traffic. One key aspect is the handling of service packets, and this is where the KUBE-SERVICES chain comes into play.



```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: demo
---
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: demo
  name: demo
  namespace: demo
spec:
  replicas: 3
  selector:
    matchLabels:
      app: demo
  template:
    metadata:
      labels:
        app: demo
    spec:
      containers:
        - image: e2eteam/echoserver:2.2-linux-arm64
          name: demo
---
apiVersion: v1
kind: Service
metadata:
  name: demo-cluster-ip-svc
  namespace: demo
spec:
  ports:
    - port: 8711
      targetPort: 8080
  selector:
    app: demo
```


<div style="text-align: center;">

![1.png](images%2F1.png)

![2.png](images%2F2.png)

</div>


**Examining the KUBE-SERVICES Chain:** KUBE-SERVICES chain is the entry point for service packets, matches the destination IP: port and dispatches the packet to the corresponding KUBE-SVC-* chain. Since KUBE-SVC-P7QE5OVRKZMQ43FN is the next chain, we will inspect that.


```shell
sudo iptables -t nat -L KUBE-SERVICES | grep demo-cluster-ip-svc
```

```log
KUBE-SVC-P7QE5OVRKZMQ43FN  tcp  --  0.0.0.0/0            10.43.169.48         /* demo/demo-cluster-ip-svc cluster IP */ tcp dpt:8711
```
The above chain rule is allowing incoming TCP traffic from anywhere to the cluster IP 10.43.169.48 on port 8711 for the Kubernetes service identified by the chain or target KUBE-SVC-P7QE5OVRKZMQ43FN.

Let's inspect the KUBE-SVC-P7QE5OVRKZMQ43FN chain.
```shell
sudo iptables -n -t nat -L KUBE-SVC-P7QE5OVRKZMQ43FN
```

```log
Chain KUBE-SVC-P7QE5OVRKZMQ43FN (1 references)
target     prot opt source               destination
KUBE-MARK-MASQ  tcp  -- !10.42.0.0/16         10.43.169.48         /* demo/demo-cluster-ip-svc cluster IP */ tcp dpt:8711
KUBE-SEP-5GSOMUMCTEFLXZNQ  all  --  0.0.0.0/0            0.0.0.0/0            /* demo/demo-cluster-ip-svc -> 10.42.1.16:8080 */ statistic mode random probability 0.33333333349
KUBE-SEP-YDSBTQTOO7DVNDMV  all  --  0.0.0.0/0            0.0.0.0/0            /* demo/demo-cluster-ip-svc -> 10.42.2.22:8080 */ statistic mode random probability 0.50000000000
KUBE-SEP-3DAK6EL2JD2M5WTS  all  --  0.0.0.0/0            0.0.0.0/0            /* demo/demo-cluster-ip-svc -> 10.42.5.23:8080 */
```
1) The source IP of the packet not comes from pod is substituted with node IP when going through chain KUBE-MARK-MASQ.
2) Then, the packet flows into any of the chain KUBE-SEP-5GSOMUMCTEFLXZNQ, KUBE-SEP-YDSBTQTOO7DVNDMV or KUBE-SEP-3DAK6EL2JD2M5WTS base on probability
```shell
sudo iptables -n -t nat -L KUBE-SEP-5GSOMUMCTEFLXZNQ
```

```log
Chain KUBE-SEP-5GSOMUMCTEFLXZNQ (1 references)
target     prot opt source               destination
KUBE-MARK-MASQ  all  --  10.42.1.16           0.0.0.0/0            /* demo/demo-cluster-ip-svc */
DNAT       tcp  --  0.0.0.0/0            0.0.0.0/0            /* demo/demo-cluster-ip-svc */ tcp to:10.42.1.16:8080
```
KUBE-SEP-5GSOMUMCTEFLXZNQ has two rules:
1. This rule is marking all traffic originating from the source IP 10.42.1.16 for masquerading. It is typically used to modify the source IP address of outgoing packets to match the node's IP address.(This is to remove circular dependency)
2. This rule is performing Destination Network Address Translation (DNAT) for incoming TCP traffic. It is redirecting incoming traffic to the node's IP address on port 8080 to the specific IP address 10.42.1.16.
