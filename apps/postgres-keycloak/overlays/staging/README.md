# Manifest Hydration

To hydrate the manifests in this repository, run the following commands:

```shell
git clone https://github.com/AuraAITools/gitops.git
# cd into the cloned directory
git checkout 31e1e8e922838211e089203f54216f1e88052cd0
kustomize build ./apps/postgres-keycloak/overlays/staging
```
