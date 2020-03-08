# Blue Green

The blue green strategy is not supported by built-in Kubernetes Deployment but available via third-party Kubernetes controller.
This example demonstrates how to implement blue-green deployment via [Argo Rollouts](https://github.com/argoproj/argo-rollouts):

1. Install Argo Rollouts controller: https://github.com/argoproj/argo-rollouts#installation
2. Create a sample application and sync it.

```
$ argocd app create --name helm-blue-green-demo --repo https://github.com/haoshuwei/argocd-samples --path blue-green --dest-server https://kubernetes.default.svc --dest-namespace blue-green --revision master
$ argocd app sync helm-blue-green-demo
```

Once the application is synced you can access it using `helm-blue-green-demo` service.

3. Change image version parameter to trigger blue-green deployment process:

```
$ argocd app set helm-blue-green-demo -p image.tag=v2 && argocd app sync helm-blue-green-demo
```

Now application runs `haoshuwei24/go-demo:v1` and `haoshuwei24/go-demo:v2` images simultaneously.
The `haoshuwei24/go-demo:v2` is still considered `blue` available only via preview service `helm-blue-green-demo-preview`.
```
$ kubectl  argo rollouts -n blue-green get rollout helm-blue-green-demo
Name:            helm-blue-green-demo
Namespace:       blue-green
Status:          ॥ Paused
Strategy:        BlueGreen
Images:          registry.cn-hangzhou.aliyuncs.com/haoshuwei24/go-demo:v1 (active)
                 registry.cn-hangzhou.aliyuncs.com/haoshuwei24/go-demo:v2 (preview)
Replicas:
  Desired:       1
  Current:       2
  Updated:       1
  Ready:         2
  Available:     1

NAME                                              KIND        STATUS     AGE    INFO
⟳ helm-blue-green-demo                            Rollout     ॥ Paused   5m1s
├──# revision:2
│  └──⧉ helm-blue-green-demo-94d8cb565            ReplicaSet  ✔ Healthy  2m43s  preview
│     └──□ helm-blue-green-demo-94d8cb565-gp7tn   Pod         ✔ Running  2m43s  ready:1/1
└──# revision:1
   └──⧉ helm-blue-green-demo-7c8b5444b6           ReplicaSet  ✔ Healthy  5m1s   active
      └──□ helm-blue-green-demo-7c8b5444b6-lf5kb  Pod         ✔ Running  5m1s   ready:1/1
```

```
$ kubectl -n blue-green get svc
NAME                           TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
helm-blue-green-demo           ClusterIP   172.27.0.140   <none>        80/TCP    6m45s
helm-blue-green-demo-preview   ClusterIP   172.27.1.4     <none>        80/TCP    6m45s
```

```
$ curl 172.27.0.140
Cluster: , Version: v1
$ curl 172.27.1.4
Cluster: , Version: v2
```

4. Promote `haoshuwei24/go-demo:v2` to `green`:

```
$ kubectl argo rollouts -n blue-green promote helm-blue-green-demo
```

```
$ kubectl  argo rollouts -n blue-green get rollout helm-blue-green-demo
Name:            helm-blue-green-demo
Namespace:       blue-green
Status:          ✔ Healthy
Strategy:        BlueGreen
Images:          registry.cn-hangzhou.aliyuncs.com/haoshuwei24/go-demo:v2 (active)
Replicas:
  Desired:       1
  Current:       1
  Updated:       1
  Ready:         1
  Available:     1

NAME                                              KIND        STATUS         AGE    INFO
⟳ helm-blue-green-demo                            Rollout     ✔ Healthy      14m
├──# revision:3
│  └──⧉ helm-blue-green-demo-64764d79ff           ReplicaSet  ✔ Healthy      2m48s  active
│     └──□ helm-blue-green-demo-64764d79ff-zwpmw  Pod         ✔ Running      2m48s  ready:1/1
├──# revision:2
│  └──⧉ helm-blue-green-demo-94d8cb565            ReplicaSet  • ScaledDown   12m
└──# revision:1
   └──⧉ helm-blue-green-demo-7c8b5444b6           ReplicaSet  • ScaledDown   14m
      └──□ helm-blue-green-demo-7c8b5444b6-lf5kb  Pod         ◌ Terminating  14m    ready:1/1
```

```
$ kubectl -n blue-green get svc
NAME                           TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
helm-blue-green-demo           ClusterIP   172.27.0.140   <none>        80/TCP    6m45s
helm-blue-green-demo-preview   ClusterIP   172.27.1.4     <none>        80/TCP    6m45s
```

```
$ curl 172.27.0.140
Cluster: , Version: v2
$ curl 172.27.1.4
Cluster: , Version: v2
```

This promotes `haoshuwei24/go-demo:v2` to `green` status and `Rollout` deletes old replica which runs `haoshuwei24/go-demo:v1`.
