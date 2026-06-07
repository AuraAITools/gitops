# Manifest Hydration

To hydrate the manifests in this repository, run the following commands:

```shell
git clone https://github.com/AuraAITools/gitops.git
# cd into the cloned directory
git checkout 2c465e50a4395f6f56e62573952d58e995ef219d
kustomize build ./apps/report-ms/overlays/dev
```
