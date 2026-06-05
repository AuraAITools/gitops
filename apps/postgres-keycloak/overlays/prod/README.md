# Manifest Hydration

To hydrate the manifests in this repository, run the following commands:

```shell
git clone https://github.com/AuraAITools/gitops.git
# cd into the cloned directory
git checkout 6747ecda0383a4a068bfcaf1be78fb3f0f9fa436
kustomize build ./apps/postgres-keycloak/overlays/prod
```
