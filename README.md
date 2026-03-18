## Commands
kubeseal -f mysecret.yaml -w mysealedsecret.yaml

kubectl get secret -n argocd argocd-secret -o yaml > secret_argocd-secret.yaml