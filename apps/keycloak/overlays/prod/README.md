# Manifest Hydration

To hydrate the manifests in this repository, run the following commands:

```shell
git clone https://github.com/AuraAITools/gitops.git
# cd into the cloned directory
git checkout 4e0d554435ad27a215f3da93fc001a374fc29eb4
kustomize build ./apps/keycloak/overlays/prod
```
