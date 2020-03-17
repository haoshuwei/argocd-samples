```
ASM operations:
$ kubectl create namespace argo-rollouts
$ kubectl apply -n argo-rollouts -f https://raw.githubusercontent.com/argoproj/argo-rollouts/stable/manifests/install.yaml
$ argocd app create --name ratings --repo https://github.com/haoshuwei/argocd-samples --path istio-bookinfo/cluster01/ratings --dest-server https://123.57.131.228:6443 --dest-namespace istio-bookinfo --revision latest
$ argocd app sync ratings
```

