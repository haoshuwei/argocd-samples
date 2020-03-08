# Blue Green

The blue green strategy is not supported by built-in Kubernetes Deployment but available via third-party Kubernetes controller.
This example demonstrates how to implement blue-green deployment via [Argo Rollouts](https://github.com/argoproj/argo-rollouts):

1. Install Argo Rollouts controller: https://github.com/argoproj/argo-rollouts#installation
2. Create a sample application and sync it.

```
argocd app create --name blue-green-demo --repo https://github.com/haoshuwei/argocd-samples --path blue-green --dest-server https://kubernetes.default.svc --dest-namespace default  && argocd app sync blue-green-demo
```

Once the application is synced you can access it using `blue-green-helm-blue-green-demo` service.

3. Change image version parameter to trigger blue-green deployment process:

```
argocd app set blue-green -p image.tag=0.2 && argocd app sync blue-green
```

Now application runs `ks-guestbook-demo:0.1` and `ks-guestbook-demo:0.2` images simultaneously.
The `ks-guestbook-demo:0.2` is still considered `blue` available only via preview service `blue-green-helm-blue-green-demo-preview`.

4. Promote `ks-guestbook-demo:0.2` to `green` by patching `Rollout` resource:

```
argocd app patch-resource blue-green --kind Rollout --resource-name blue-green-helm-blue-green-demo --patch '{ "status": { "verifyingPreview": false } }' --patch-type 'application/merge-patch+json'
```

This promotes `ks-guestbook-demo:0.2` to `green` status and `Rollout` deletes old replica which runs `ks-guestbook-demo:0.1`.
