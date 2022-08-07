# SimplePhone-deploy

## Setup CD (flux bootstrap)

1. Install flux CD:

MacOS:
`brew install fluxcd/tap/flux`

For other installation options, see [Installing the Flux CLI](https://toolkit.fluxcd.io/get-started/#install-the-flux-cli).

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