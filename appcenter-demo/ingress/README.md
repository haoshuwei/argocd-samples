```bash
$ appcenter repo add https://github.com/haoshuwei/argocd-samples.git --type git --name git-repo-ingress
$ appcenter app create --name git-ingress-demo --repo https://github.com/haoshuwei/argocd-samples.git --revision latest --path appcenter-demo/ingress --dest-namespace canary --dest-server https://kubernetes.default.svc
```