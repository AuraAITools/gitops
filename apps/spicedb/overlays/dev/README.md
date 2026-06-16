# Manifest Hydration

To hydrate the manifests in this repository, run the following commands:

```shell
git clone https://github.com/AuraAITools/gitops.git
# cd into the cloned directory
git checkout 5b75ab65a8fd87e3354b5d946856a6fe21c38af8
kustomize build ./apps/spicedb/overlays/dev
```
