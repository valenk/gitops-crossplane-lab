# Setup

1. Cluster
```bash
k3d cluster create xp-argo --k3s-arg "--disable=traefik@server:0"
```

2. floci (a AWS local)
```bash
sudo podman start floci floci-aws-ui
alias awsf='aws --profile floci --endpoint-url http://localhost:4566'
```

3. Push do repo
```bash
cd ~/lab-crossplane/v2/gitops-crossplane-lab
git add -A && git commit -m "lab" && git push
```

4. Instalar o Argo CD (Helm)
```bash
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update
helm upgrade --install argocd argo/argo-cd --version 10.11.6 \
  -n argocd --create-namespace -f argocd/values.yaml --wait --timeout 5m
```

5. Secret do floci
```bash
kubectl create namespace crossplane-system
kubectl create secret generic aws-secret -n crossplane-system \
  --from-literal=creds='[default]
aws_access_key_id = test
aws_secret_access_key = test'
```

6. Arrancar o GitOps
```bash
kubectl apply -f bootstrap/root.yaml
```

7. Ver as apps a subir (Ctrl+C para sair)
```bash
kubectl get applications -n argocd -w
```

Só se aparecer `no such host` nos eventos:
```bash
kubectl get cm coredns -n kube-system -o yaml \
  | sed 's#forward . /etc/resolv.conf#forward . 1.1.1.1 8.8.8.8#' | kubectl replace -f -
kubectl rollout restart deploy/coredns -n kube-system
```

# UI do Argo

1. Password do `admin`
```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath='{.data.password}' | base64 -d; echo
```

2. Port-forward (deixa a correr) e abre `https://localhost:8080`
```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

# Test

1. Está tudo pronto?
```bash
kubectl get applications -n argocd
kubectl get providers,functions
kubectl get xrd,compositions
```

2. O bucket existe (cluster e floci)?
```bash
kubectl get storagebuckets.platform.example.org -n default
kubectl get buckets.s3.aws.m.upbound.io -n default
awsf s3 ls
```

3. Bucket novo = ficheiro novo no Git
```bash
sed 's/my-bucket/other-bucket/' consumers/default/my-bucket.yaml > consumers/default/other-bucket.yaml
git add -A && git commit -m "other-bucket" && git push
kubectl annotate application consumers -n argocd argocd.argoproj.io/refresh=normal --overwrite
awsf s3 ls
```

4. Apagar = tirar do Git (prune)
```bash
git rm consumers/default/other-bucket.yaml
git commit -m "remove other-bucket" && git push
kubectl annotate application consumers -n argocd argocd.argoproj.io/refresh=normal --overwrite
awsf s3 ls
```

5. Apagar à mão um recurso do Git: o Argo recria (selfHeal)
```bash
kubectl get storagebuckets.platform.example.org -n default -w --output-watch-events &
kubectl delete -f consumers/default/my-bucket.yaml
kubectl get events -n argocd --sort-by=.lastTimestamp | grep consumers | tail -5
```

6. Criar à mão um recurso fora do Git: o Argo ignora
```bash
sed 's/my-bucket/manual-bucket/' consumers/default/my-bucket.yaml | kubectl apply -f -
kubectl get storagebuckets.platform.example.org -n default
kubectl delete storagebuckets.platform.example.org manual-bucket -n default
```

7. Apagar o bucket no floci: o Crossplane recria
```bash
awsf s3 rb s3://my-bucket
awsf s3 ls
```

# Limpar

```bash
kubectl delete application root -n argocd
kubectl delete storagebuckets.platform.example.org --all -A
awsf s3 ls
k3d cluster delete xp-argo
```
