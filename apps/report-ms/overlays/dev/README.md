# Manifest Hydration

To hydrate the manifests in this repository, run the following commands:

```shell
git clone https://github.com/AuraAITools/gitops.git
# cd into the cloned directory
git checkout 56d5ecfa474f104df9ab258fc8881d0523a9d945
kustomize build ./apps/report-ms/overlays/dev
```
