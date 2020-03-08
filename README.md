# argocd-samples

This is a Kustomized application demo deployed in multiple clusters/clouds. We will deploy
one application in 3 clusters with kustomized pvc and env.

#### 1 Add 3 clusters with kubeconfig file: ack-pre, ack-pro and gke-pro
```
$ argocd cluster list
SERVER                          NAME     VERSION  STATUS      MESSAGE
https://xxx.xx.xxx.xxx:6443     ack-pro  1.14+    Successful
https://xx.xx.xxx.xxx           gke-pro  1.14+    Successful
https://xxx.xx.xxx.xxx:6443     ack-pre  1.14+    Successful
https://kubernetes.default.svc           1.14+    Successful
```

#### 2 Create ack-pre
Create ack-pre in cluster ack-pre.
Automatically sync repo `https://github.com/haoshuwei/argocd-samples.git` branch `latest`, and deploy resources declared in path `overlays/pre` to cluster `https://xx.xx.xxx.xxx:6443` namespace `argocd-samples`
```
$ argocd app create --project default --name ack-pre --repo https://github.com/haoshuwei/argocd-samples.git --path overlays/pre --dest-server https://xx.xx.xxx.xxx:6443 --dest-namespace  argocd-samples --revision latest --sync-policy automated
```

```
$ kubectl -n argocd-samples get svc
NAME       TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
app-demo   ClusterIP   172.27.8.141   <none>        80/TCP    17h

$ curl 172.27.8.141
Cluster: ACK Pre, Version: v1
```

#### 3 Create ack-pro
Create ack-pro in cluster ack-pro.
Manually sync repo `https://github.com/haoshuwei/argocd-samples.git` branch `master`, and deploy resources declared in path `overlays/pro` to cluster `https://xx.xx.xxx.xxx:6443` namespace `argocd-samples`
```
$ argocd app create --project default --name ack-pro --repo https://github.com/haoshuwei/argocd-samples.git --path overlays/pro --dest-server https://xx.xx.xxx.xxx:6443 --dest-namespace  argocd-samples --revision master
```

```
$ kubectl -n argocd-samples get svc
NAME       TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
app-demo   ClusterIP   172.23.8.250   <none>        80/TCP    16h
$ curl 172.23.8.250
Cluster: ACK Pro, Version: v1
```

#### 4 Create gke-pro
Create gke-pro in cluster gke-pro.
Manually sync repo `https://github.com/haoshuwei/argocd-samples.git` branch `master`, and deploy resources declared in path `overlays/gke` to cluster `https://xx.xx.xxx.xxx` namespace `argocd-samples`
```
$ argocd app create --project default --name gke-pro --repo https://github.com/haoshuwei/argocd-samples.git --path overlays/gke --dest-server https://xx.xx.xxx.xxx --dest-namespace  argocd-samples --revision master
```

```
$ kubectl -n argocd-samples get svc
NAME       TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
app-demo   ClusterIP   172.21.10.31   <none>        80/TCP    14h
$ curl 172.21.10.31
Cluster: GKE Pro, Version: v1
```

#### 5 GitOps process


# blue-green
Details are in [Blue-Green](https://github.com/haoshuwei/argocd-samples/blob/master/blue-green/README.md)

# canary
Details are in [Canary](https://github.com/haoshuwei/argocd-samples/blob/master/canary/README.md)