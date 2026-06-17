# Manifest Hydration

To hydrate the manifests in this repository, run the following commands:

```shell
git clone https://github.com/AuraAITools/gitops.git
# cd into the cloned directory
git checkout 21354a6b55db0fd41d236fe0c8738b4b50c38e1d
kustomize build ./apps/report-ms/overlays/dev
```
