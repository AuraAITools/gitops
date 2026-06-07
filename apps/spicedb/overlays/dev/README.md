# Manifest Hydration

To hydrate the manifests in this repository, run the following commands:

```shell
git clone https://github.com/AuraAITools/gitops.git
# cd into the cloned directory
git checkout 06cf32a23e6ba2dec7a94f6ffbf35e8e319ecd7e
kustomize build ./apps/spicedb/overlays/dev
```
