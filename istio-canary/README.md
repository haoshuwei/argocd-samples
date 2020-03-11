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

Once the application is synced you can access it using `istio-ingressgateway` service.
```
$ kubectl -n istio-system get svc|grep istio-ingressgateway
  istio-ingressgateway     LoadBalancer   172.27.1.78     xx.xxx.xxx.xxx   15020:31944/TCP,80:31627/TCP,443:32212/TCP,15443:30574/TCP   15h
```

```
$ while sleep 0.5; do curl xx.xxx.xxx.xxx; done
Cluster: , Version: v1
Cluster: , Version: v1
Cluster: , Version: v1
Cluster: , Version: v1
Cluster: , Version: v1
```

The canary strategy:
```
strategy:
  canary:
    steps:
    - setWeight: 20
    - pause:
        {}
    - setWeight: 40
    - pause:
        {duration: 40}
    - setWeight: 60
    - pause:
        {}
    - setWeight: 80
    - pause:
        {}
    canaryService: canary-svc # required
    stableService: stable-svc  # required
    trafficRouting:
      istio:
        virtualService:
          name: rollout-vsvc  # required
          routes:
          - primary # At least one route is required
```

Argo Rollouts will promote app to setWeight=20,40,80,100 manually, setWeight=60 automatically by updating istio virtualService.

Current helm-istio-canary-demo app status:
```
$ kubectl argo rollouts -n istio get rollout helm-istio-canary-demo
kubectl argo rollouts -n istio get rollout helm-istio-canary-demo
Name:            helm-istio-canary-demo
Namespace:       istio
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

NAME                                                KIND        STATUS     AGE  INFO
⟳ helm-istio-canary-demo                            Rollout     ✔ Healthy  19s
└──# revision:1
   └──⧉ helm-istio-canary-demo-67b4d4dc89           ReplicaSet  ✔ Healthy  19s  stable
      ├──□ helm-istio-canary-demo-67b4d4dc89-4sh7s  Pod         ✔ Running  19s  ready:2/2
      ├──□ helm-istio-canary-demo-67b4d4dc89-5trjs  Pod         ✔ Running  19s  ready:2/2
      ├──□ helm-istio-canary-demo-67b4d4dc89-8vgqd  Pod         ✔ Running  19s  ready:2/2
      ├──□ helm-istio-canary-demo-67b4d4dc89-t9ghj  Pod         ✔ Running  19s  ready:2/2
      └──□ helm-istio-canary-demo-67b4d4dc89-xttpx  Pod         ✔ Running  19s  ready:2/2
```

```
$ while sleep 0.5; do curl xx.xxx.xxx.xxx; done
Cluster: , Version: v1
Cluster: , Version: v1
Cluster: , Version: v1
Cluster: , Version: v1
Cluster: , Version: v1
```

#### 3 Change image version parameter to trigger canary deployment process:

Change image version parameter and sync the app:
```
$ argocd app set helm-istio-canary-demo -p image.tag=v2 && argocd app sync helm-istio-canary-demo
```
`ActualWeight:  20`:
```
Name:            helm-istio-canary-demo
Namespace:       istio
Status:          ॥ Paused
Strategy:        Canary
  Step:          1/8
  SetWeight:     20
  ActualWeight:  16
Images:          registry.cn-hangzhou.aliyuncs.com/haoshuwei24/go-demo:v1 (stable)
                 registry.cn-hangzhou.aliyuncs.com/haoshuwei24/go-demo:v2 (canary)
Replicas:
  Desired:       5
  Current:       6
  Updated:       1
  Ready:         6
  Available:     6

NAME                                                KIND        STATUS     AGE  INFO
⟳ helm-istio-canary-demo                            Rollout     ॥ Paused   43m
├──# revision:2
│  └──⧉ helm-istio-canary-demo-59689777db           ReplicaSet  ✔ Healthy  20s  canary
│     └──□ helm-istio-canary-demo-59689777db-9m52r  Pod         ✔ Running  20s  ready:2/2
└──# revision:1
   └──⧉ helm-istio-canary-demo-67b4d4dc89           ReplicaSet  ✔ Healthy  43m  stable
      ├──□ helm-istio-canary-demo-67b4d4dc89-4sh7s  Pod         ✔ Running  43m  ready:2/2
      ├──□ helm-istio-canary-demo-67b4d4dc89-5trjs  Pod         ✔ Running  43m  ready:2/2
      ├──□ helm-istio-canary-demo-67b4d4dc89-8vgqd  Pod         ✔ Running  43m  ready:2/2
      ├──□ helm-istio-canary-demo-67b4d4dc89-t9ghj  Pod         ✔ Running  43m  ready:2/2
      └──□ helm-istio-canary-demo-67b4d4dc89-xttpx  Pod         ✔ Running  43m  ready:2/2
```

Check `virtualservice rollout-vsvc`:
```
  http:
  - name: primary
    route:
    - destination:
        host: stable-svc
      weight: 80
    - destination:
        host: canary-svc
      weight: 20
```

```
$ while sleep 0.5; do curl xx.xxx.xxx.xxx; done
Cluster: , Version: v1
Cluster: , Version: v1
Cluster: , Version: v1
Cluster: , Version: v1
Cluster: , Version: v1
Cluster: , Version: v2
Cluster: , Version: v1
Cluster: , Version: v1
```

#### 4 Promote `haoshuwei24/go-demo:v2` to weight `40` and auto to `60`:

```
$ kubectl argo rollouts -n istio promote helm-istio-canary-demo
```

```
$ kubectl argo rollouts -n istio get rollout helm-istio-canary-demo
  Name:            helm-istio-canary-demo
  Namespace:       istio
  Status:          ॥ Paused
  Strategy:        Canary
    Step:          3/8
    SetWeight:     40
    ActualWeight:  28
  Images:          registry.cn-hangzhou.aliyuncs.com/haoshuwei24/go-demo:v1 (stable)
                   registry.cn-hangzhou.aliyuncs.com/haoshuwei24/go-demo:v2 (canary)
  Replicas:
    Desired:       5
    Current:       7
    Updated:       2
    Ready:         7
    Available:     7
  
  NAME                                                KIND        STATUS     AGE    INFO
  ⟳ helm-istio-canary-demo                            Rollout     ॥ Paused   48m
  ├──# revision:2
  │  └──⧉ helm-istio-canary-demo-59689777db           ReplicaSet  ✔ Healthy  5m29s  canary
  │     ├──□ helm-istio-canary-demo-59689777db-9m52r  Pod         ✔ Running  5m29s  ready:2/2
  │     └──□ helm-istio-canary-demo-59689777db-mxmzn  Pod         ✔ Running  43s    ready:2/2
  └──# revision:1
     └──⧉ helm-istio-canary-demo-67b4d4dc89           ReplicaSet  ✔ Healthy  48m    stable
        ├──□ helm-istio-canary-demo-67b4d4dc89-4sh7s  Pod         ✔ Running  48m    ready:2/2
        ├──□ helm-istio-canary-demo-67b4d4dc89-5trjs  Pod         ✔ Running  48m    ready:2/2
        ├──□ helm-istio-canary-demo-67b4d4dc89-8vgqd  Pod         ✔ Running  48m    ready:2/2
        ├──□ helm-istio-canary-demo-67b4d4dc89-t9ghj  Pod         ✔ Running  48m    ready:2/2
        └──□ helm-istio-canary-demo-67b4d4dc89-xttpx  Pod         ✔ Running  48m    ready:2/2
```

```
$ kubectl argo rollouts -n istio get rollout helm-istio-canary-demo
Name:            helm-istio-canary-demo
Namespace:       istio
Status:          ॥ Paused
Strategy:        Canary
  Step:          5/8
  SetWeight:     60
  ActualWeight:  37
Images:          registry.cn-hangzhou.aliyuncs.com/haoshuwei24/go-demo:v1 (stable)
                 registry.cn-hangzhou.aliyuncs.com/haoshuwei24/go-demo:v2 (canary)
Replicas:
  Desired:       5
  Current:       8
  Updated:       3
  Ready:         8
  Available:     8

NAME                                                KIND        STATUS     AGE    INFO
⟳ helm-istio-canary-demo                            Rollout     ॥ Paused   49m
├──# revision:2
│  └──⧉ helm-istio-canary-demo-59689777db           ReplicaSet  ✔ Healthy  7m16s  canary
│     ├──□ helm-istio-canary-demo-59689777db-9m52r  Pod         ✔ Running  7m16s  ready:2/2
│     ├──□ helm-istio-canary-demo-59689777db-mxmzn  Pod         ✔ Running  2m30s  ready:2/2
│     └──□ helm-istio-canary-demo-59689777db-qn2dx  Pod         ✔ Running  99s    ready:2/2
└──# revision:1
   └──⧉ helm-istio-canary-demo-67b4d4dc89           ReplicaSet  ✔ Healthy  49m    stable
      ├──□ helm-istio-canary-demo-67b4d4dc89-4sh7s  Pod         ✔ Running  49m    ready:2/2
      ├──□ helm-istio-canary-demo-67b4d4dc89-5trjs  Pod         ✔ Running  49m    ready:2/2
      ├──□ helm-istio-canary-demo-67b4d4dc89-8vgqd  Pod         ✔ Running  49m    ready:2/2
      ├──□ helm-istio-canary-demo-67b4d4dc89-t9ghj  Pod         ✔ Running  49m    ready:2/2
      └──□ helm-istio-canary-demo-67b4d4dc89-xttpx  Pod         ✔ Running  49m    ready:2/2
```
Check `virtualservice rollout-vsvc`:
```
http:
  - name: primary
    route:
    - destination:
        host: stable-svc
      weight: 40
    - destination:
        host: canary-svc
      weight: 60
``` 

#### 5 Promote `haoshuwei24/go-demo:v2` to weight `80`
```
$ kubectl argo rollouts -n istio promote helm-istio-canary-demo
```

```
$ kubectl argo rollouts -n istio get rollout helm-istio-canary-demo
Name:            helm-istio-canary-demo
Namespace:       istio
Status:          ॥ Paused
Strategy:        Canary
  Step:          7/8
  SetWeight:     80
  ActualWeight:  44
Images:          registry.cn-hangzhou.aliyuncs.com/haoshuwei24/go-demo:v1 (stable)
                 registry.cn-hangzhou.aliyuncs.com/haoshuwei24/go-demo:v2 (canary)
Replicas:
  Desired:       5
  Current:       9
  Updated:       4
  Ready:         9
  Available:     9

NAME                                                KIND        STATUS     AGE    INFO
⟳ helm-istio-canary-demo                            Rollout     ॥ Paused   56m
├──# revision:2
│  └──⧉ helm-istio-canary-demo-59689777db           ReplicaSet  ✔ Healthy  13m    canary
│     ├──□ helm-istio-canary-demo-59689777db-9m52r  Pod         ✔ Running  13m    ready:2/2
│     ├──□ helm-istio-canary-demo-59689777db-mxmzn  Pod         ✔ Running  8m37s  ready:2/2
│     ├──□ helm-istio-canary-demo-59689777db-qn2dx  Pod         ✔ Running  7m46s  ready:2/2
│     └──□ helm-istio-canary-demo-59689777db-26fh5  Pod         ✔ Running  34s    ready:2/2
└──# revision:1
   └──⧉ helm-istio-canary-demo-67b4d4dc89           ReplicaSet  ✔ Healthy  56m    stable
      ├──□ helm-istio-canary-demo-67b4d4dc89-4sh7s  Pod         ✔ Running  56m    ready:2/2
      ├──□ helm-istio-canary-demo-67b4d4dc89-5trjs  Pod         ✔ Running  56m    ready:2/2
      ├──□ helm-istio-canary-demo-67b4d4dc89-8vgqd  Pod         ✔ Running  56m    ready:2/2
      ├──□ helm-istio-canary-demo-67b4d4dc89-t9ghj  Pod         ✔ Running  56m    ready:2/2
      └──□ helm-istio-canary-demo-67b4d4dc89-xttpx  Pod         ✔ Running  56m    ready:2/2
```

#### 6 Promote `haoshuwei24/go-demo:v2` to weight `100`
```
$ kubectl argo rollouts -n istio promote helm-istio-canary-demo
```
```
$ kubectl argo rollouts -n istio get rollout helm-istio-canary-demo
  Name:            helm-istio-canary-demo
  Namespace:       istio
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
  
  NAME                                                KIND        STATUS        AGE    INFO
  ⟳ helm-istio-canary-demo                            Rollout     ✔ Healthy     61m
  ├──# revision:2
  │  └──⧉ helm-istio-canary-demo-59689777db           ReplicaSet  ✔ Healthy     18m    stable
  │     ├──□ helm-istio-canary-demo-59689777db-9m52r  Pod         ✔ Running     18m    ready:2/2
  │     ├──□ helm-istio-canary-demo-59689777db-mxmzn  Pod         ✔ Running     13m    ready:2/2
  │     ├──□ helm-istio-canary-demo-59689777db-qn2dx  Pod         ✔ Running     12m    ready:2/2
  │     ├──□ helm-istio-canary-demo-59689777db-26fh5  Pod         ✔ Running     5m47s  ready:2/2
  │     └──□ helm-istio-canary-demo-59689777db-dnfbw  Pod         ✔ Running     66s    ready:2/2
  └──# revision:1
     └──⧉ helm-istio-canary-demo-67b4d4dc89           ReplicaSet  • ScaledDown  61m
```

```
$ while sleep 0.5; do curl xx.xxx.xxx.xxx; done
Cluster: , Version: v2
Cluster: , Version: v2
Cluster: , Version: v2
Cluster: , Version: v2
Cluster: , Version: v2
```
