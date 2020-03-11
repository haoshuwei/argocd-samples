# Canary with Istio

The canary strategy is not supported by built-in Kubernetes Deployment but available via third-party Kubernetes controller.
This example demonstrates how to implement blue-green deployment via [Argo Rollouts](https://github.com/argoproj/argo-rollouts):

#### 1 Install Argo Rollouts controller 
[https://github.com/argoproj/argo-rollouts#installation](https://github.com/argoproj/argo-rollouts#installation)
#### 2 Create a sample application and sync it.

```
$ argocd app create --name helm-istio-canary-demo --repo https://github.com/haoshuwei/argocd-samples --path istio-canary --dest-server https://kubernetes.default.svc --dest-namespace istio --revision latest
$ argocd app sync helm-istio-canary-demo
```

Once the application is synced you can access it using `hhelm-istio-canary-demo` service.

```
$ kubectl argo rollouts -n istio get rollout helm-istio-canary-demo
Name:            helm-istio-canary-demo
Namespace:       canary
Status:          ✔ Healthy
Strategy:        Canary
  Step:          8/8
  SetWeight:     100
  ActualWeight:  100
Images:          registry.cn-hangzhou.aliyuncs.com/haoshuwei24/go-demo:v1 (stable)
Replicas:
  Desired:       5
  Current:       5
  Updated:       5
  Ready:         5
  Available:     5

NAME                                          KIND        STATUS     AGE    INFO
⟳ helm-istio-canary-demo                            Rollout     ✔ Healthy  7m46s
└──# revision:1
   └──⧉ helm-istio-canary-demo-5577cb9b4b           ReplicaSet  ✔ Healthy  7m46s  stable
      ├──□ helm-istio-canary-demo-5577cb9b4b-9qs8g  Pod         ✔ Running  7m46s  ready:1/1
      ├──□ helm-istio-canary-demo-5577cb9b4b-fmrmq  Pod         ✔ Running  7m46s  ready:1/1
      ├──□ helm-istio-canary-demo-5577cb9b4b-gkz2g  Pod         ✔ Running  7m46s  ready:1/1
      ├──□ helm-istio-canary-demo-5577cb9b4b-qh7t4  Pod         ✔ Running  7m46s  ready:1/1
      └──□ helm-istio-canary-demo-5577cb9b4b-tgtjw  Pod         ✔ Running  7m46s  ready:1/1
```

```
$ kubectl -n canary get svc
NAME                       TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
helm-istio-canary-demo           ClusterIP   172.27.0.160   <none>        80/TCP    22s
```

```
$ while sleep 0.5; do curl 172.27.0.160; done
Cluster: , Version: v1
Cluster: , Version: v1
Cluster: , Version: v1
Cluster: , Version: v1
Cluster: , Version: v1
```

#### 3 Change image version parameter to trigger canary deployment process:

The canary strategy means that canary deployment will promote manully when setWeight from 20 to 40 and from 80 to 100:
```
strategy:
  canary:
    steps:
    - setWeight: 20
    - pause: {}
    - setWeight: 40
    - pause: {duration: 40}
    - setWeight: 60
    - pause: {duration: 20}
    - setWeight: 80
    - pause: {}
```

Change image version parameter and sync the app:
```
$ argocd app set helm-istio-canary-demo -p image.tag=v2 && argocd app sync helm-istio-canary-demo
```
`ActualWeight:  20`:
```
$ kubectl argo rollouts -n canary get rollout helm-istio-canary-demo
Name:            helm-istio-canary-demo
Namespace:       canary
Status:          ॥ Paused
Strategy:        Canary
  Step:          1/8
  SetWeight:     20
  ActualWeight:  20
Images:          registry.cn-hangzhou.aliyuncs.com/haoshuwei24/go-demo:v1 (stable)
                 registry.cn-hangzhou.aliyuncs.com/haoshuwei24/go-demo:v2 (canary)
Replicas:
  Desired:       5
  Current:       5
  Updated:       1
  Ready:         5
  Available:     5

NAME                                          KIND        STATUS     AGE    INFO
⟳ helm-istio-canary-demo                            Rollout     ॥ Paused   12m
├──# revision:2
│  └──⧉ helm-istio-canary-demo-79894f78f            ReplicaSet  ✔ Healthy  3m56s  canary
│     └──□ helm-istio-canary-demo-79894f78f-2dgwb   Pod         ✔ Running  3m56s  ready:1/1
└──# revision:1
   └──⧉ helm-istio-canary-demo-5577cb9b4b           ReplicaSet  ✔ Healthy  12m    stable
      ├──□ helm-istio-canary-demo-5577cb9b4b-9qs8g  Pod         ✔ Running  12m    ready:1/1
      ├──□ helm-istio-canary-demo-5577cb9b4b-fmrmq  Pod         ✔ Running  12m    ready:1/1
      ├──□ helm-istio-canary-demo-5577cb9b4b-qh7t4  Pod         ✔ Running  12m    ready:1/1
      └──□ helm-istio-canary-demo-5577cb9b4b-tgtjw  Pod         ✔ Running  12m    ready:1/1
```

```
$ kubectl -n canary get svc
NAME                       TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
helm-istio-canary-demo           ClusterIP   172.27.0.160   <none>        80/TCP    22s
```

```
$ while sleep 0.5; do curl 172.27.0.160; done
Cluster: , Version: v2
Cluster: , Version: v1
Cluster: , Version: v1
Cluster: , Version: v1
Cluster: , Version: v1
```

#### 4 Promote `haoshuwei24/go-demo:v2` to weight `40`, then automatically to `60` and `80`:

```
$ kubectl argo rollouts -n canary promote helm-istio-canary-demo
```

```
$ kubectl argo rollouts -n canary get rollout helm-istio-canary-demo
Name:            helm-istio-canary-demo
Namespace:       canary
Status:          ॥ Paused
Strategy:        Canary
  Step:          7/8
  SetWeight:     80
  ActualWeight:  80
Images:          registry.cn-hangzhou.aliyuncs.com/haoshuwei24/go-demo:v1 (stable)
                 registry.cn-hangzhou.aliyuncs.com/haoshuwei24/go-demo:v2 (canary)
Replicas:
  Desired:       5
  Current:       5
  Updated:       4
  Ready:         5
  Available:     5

NAME                                          KIND        STATUS     AGE    INFO
⟳ helm-istio-canary-demo                            Rollout     ॥ Paused   18m
├──# revision:2
│  └──⧉ helm-istio-canary-demo-79894f78f            ReplicaSet  ✔ Healthy  9m20s  canary
│     ├──□ helm-istio-canary-demo-79894f78f-2dgwb   Pod         ✔ Running  9m20s  ready:1/1
│     ├──□ helm-istio-canary-demo-79894f78f-lzqhr   Pod         ✔ Running  2m3s   ready:1/1
│     ├──□ helm-istio-canary-demo-79894f78f-rwxtl   Pod         ✔ Running  79s    ready:1/1
│     └──□ helm-istio-canary-demo-79894f78f-xzcmq   Pod         ✔ Running  55s    ready:1/1
└──# revision:1
   └──⧉ helm-istio-canary-demo-5577cb9b4b           ReplicaSet  ✔ Healthy  18m    stable
      └──□ helm-istio-canary-demo-5577cb9b4b-qh7t4  Pod         ✔ Running  18m    ready:1/1
```

```
$ kubectl -n canary get svc
NAME                       TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
helm-istio-canary-demo           ClusterIP   172.27.0.160   <none>        80/TCP    22s
```

```
$ while sleep 0.5; do curl 172.27.0.160; done
Cluster: , Version: v1
Cluster: , Version: v2
Cluster: , Version: v2
Cluster: , Version: v2
Cluster: , Version: v2
```

#### 5 Promote `haoshuwei24/go-demo:v2` to weight `100`
```
$ kubectl argo rollouts -n canary promote helm-istio-canary-demo
```

```
$ kubectl argo rollouts -n canary get rollout helm-istio-canary-demo
Name:            helm-istio-canary-demo
Namespace:       canary
Status:          ✔ Healthy
Strategy:        Canary
  Step:          8/8
  SetWeight:     100
  ActualWeight:  100
Images:          registry.cn-hangzhou.aliyuncs.com/haoshuwei24/go-demo:v2 (stable)
Replicas:
  Desired:       5
  Current:       5
  Updated:       5
  Ready:         5
  Available:     5

NAME                                         KIND        STATUS        AGE    INFO
⟳ helm-istio-canary-demo                           Rollout     ✔ Healthy     21m
├──# revision:2
│  └──⧉ helm-istio-canary-demo-79894f78f           ReplicaSet  ✔ Healthy     12m    stable
│     ├──□ helm-istio-canary-demo-79894f78f-2dgwb  Pod         ✔ Running     12m    ready:1/1
│     ├──□ helm-istio-canary-demo-79894f78f-lzqhr  Pod         ✔ Running     5m15s  ready:1/1
│     ├──□ helm-istio-canary-demo-79894f78f-rwxtl  Pod         ✔ Running     4m31s  ready:1/1
│     ├──□ helm-istio-canary-demo-79894f78f-xzcmq  Pod         ✔ Running     4m7s   ready:1/1
│     └──□ helm-istio-canary-demo-79894f78f-pjg5k  Pod         ✔ Running     41s    ready:1/1
└──# revision:1
   └──⧉ helm-istio-canary-demo-5577cb9b4b          ReplicaSet  • ScaledDown  21m
```

```
$ kubectl -n canary get svc
NAME                       TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
helm-istio-canary-demo           ClusterIP   172.27.0.160   <none>        80/TCP    22s
```

```
$ while sleep 0.5; do curl 172.27.0.160; done
Cluster: , Version: v2
Cluster: , Version: v2
Cluster: , Version: v2
Cluster: , Version: v2
Cluster: , Version: v2
```
