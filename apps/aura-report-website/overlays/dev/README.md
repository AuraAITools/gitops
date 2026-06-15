# Manifest Hydration

To hydrate the manifests in this repository, run the following commands:

```shell
git clone https://github.com/AuraAITools/gitops.git
# cd into the cloned directory
git checkout f18d9b7fc0ba93522c44dd43a6ea2d2b9f951e0d
kustomize build ./apps/aura-report-website/overlays/dev
```
