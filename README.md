# SimplePhone-deploy

1. Install [flux CD](https://toolkit.fluxcd.io/get-started/#install-the-flux-cli)
2. Install [kind](https://kind.sigs.k8s.io/docs/user/quick-start/#installation)
3. Install [helm](https://helm.sh/docs/intro/install/)

## Development setup

```sh
cd kind
kind create cluster --config kind.yaml --name dev
kind export kubeconfig --name dev
kubectl config use-context kind-dev
make dev

# Access to application by port forwarding
kubectl port-forward `kubectl get pod -l run=simplephone --no-headers -o custom-columns=":metadata.name"` 8080:80
# Browse http://127.0.0.1:8080/
```

### Clean Up
```sh
make clean
```

## Production setup

1. Connect to production k8s
```sh
kubectl config set-context <K8S Cluster Context>
```

2. Once installed, check that your EKS/k8s cluster satisfies the prerequisites:

`flux check --pre`

If successful, it returns something similar to this:
```
► checking prerequisites
✔ Kubernetes 1.22.0 >=1.20.6-0
✔ prerequisites checks passed
```

3. Bootstrap the Flux Config Repository:
```
export GITHUB_TOKEN=<personal_access_token>
export GITHUB_USER=<github_username>
export GITHUB_REPO=SimplePhone-deploy

flux bootstrap github \
    --owner="${GITHUB_USER}" \
    --repository="${GITHUB_REPO}" \
    --private=false \
    --path=./clusters/cluster1 \
    --personal \
    --token-auth
```

4. Create a flux secret
```
flux create secret git simplephone-flux-secret \
    --url=ssh://git@github.com/${GITHUB_USER}/${GITHUB_REPO}
```

4. Add Deploy key into Github repo(Settings -> Deploy Keys):
```
kubectl get secret simplephone-flux-secret -n flux-system -ojson \
    | jq -r '.data."identity.pub"' | base64 -d
```

5. Create a flux source:
```
git clone git@github.com:afshinpaydar-binary/SimplePhone-deploy.git
cd SimplePhone-deploy

flux create source git simplephone \
    --url=ssh://git@github.com/${GITHUB_USER}/${GITHUB_REPO} \
    --secret-ref simplephone-flux-secret \
    --branch=main \
    --export > ./clusters/cluster1/simplephone-flux-source.yaml

# Verify
flux get source git
cat ./clusters/cluster1/simplephone-flux-source.yaml
```

6. Create flux kustomization and push changes into deployment repo(GitOps):
```
flux create kustomization simplephone \
  --source=simplephone \
  --path=./eks-prod \
  --prune=true \
  --interval=1m \
  --export > ./clusters/cluster1/simplephone-kustomization.yaml

git add ./clusters/cluster1/simplephone-flux-source.yaml ./clusters/cluster1/simplephone-kustomization.yaml
git commit -m "add simplephone source"
git push origin main

# verify
flux reconcile source git simplephone
watch kubectl get -n flux-system gitrepositories
```

7. Watch the Kustomization
```
watch flux get kustomizations
```