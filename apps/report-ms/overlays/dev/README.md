# Manifest Hydration

To hydrate the manifests in this repository, run the following commands:

```shell
git clone https://github.com/AuraAITools/gitops.git
# cd into the cloned directory
git checkout 2e7bb931a52927a588cfd4be417e15dd3b040cc8
kustomize build ./apps/report-ms/overlays/dev
```
