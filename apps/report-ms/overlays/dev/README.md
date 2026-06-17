# Manifest Hydration

To hydrate the manifests in this repository, run the following commands:

```shell
git clone https://github.com/AuraAITools/gitops.git
# cd into the cloned directory
git checkout e9cc58d07f1ecc0fcc7776d98b9634fa7fbf1fff
kustomize build ./apps/report-ms/overlays/dev
```
