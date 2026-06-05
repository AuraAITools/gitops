# Manifest Hydration

To hydrate the manifests in this repository, run the following commands:

```shell
git clone https://github.com/AuraAITools/gitops.git
# cd into the cloned directory
git checkout be68fed8669b04aa623b7ed77a825f832df1d1fe
kustomize build ./apps/report-ms/overlays/dev
```
