```
argocd app create --project default --name details --repo https://github.com/haoshuwei/argocd-samples.git --path tracing/istio-bookinfo/details  --dest-server https://xxx.xx.xxx.xxx:6443 --dest-namespace bookinfo --revision latest --sync-policy automated
argocd app create --project default --name ratings --repo https://github.com/haoshuwei/argocd-samples.git --path tracing/istio-bookinfo/ratings  --dest-server https://115.29.211.115:6443 --dest-namespace bookinfo --revision latest --sync-policy automated
argocd app create --project default --name reviews --repo https://github.com/haoshuwei/argocd-samples.git --path tracing/istio-bookinfo/reviews  --dest-server https://115.29.211.115:6443 --dest-namespace bookinfo --revision latest --sync-policy automated
argocd app create --project default --name productpage --repo https://github.com/haoshuwei/argocd-samples.git --path tracing/istio-bookinfo/productpage  --dest-server https://115.29.211.115:6443 --dest-namespace bookinfo --revision latest --sync-policy automated
```


```
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: bookinfo
spec:
  hosts:
  - bookinfo.istio.example.com
  gateways:
  - public-gateway.istio-system.svc.cluster.local
  http:
  - match:
    - uri:
        exact: /productpage
    - uri:
        prefix: /static
    - uri:
        exact: /login
    - uri:
        exact: /logout
    - uri:
        prefix: /api/v1/products
    route:
    - destination:
        host: productpage
        port:
          number: 9080
```