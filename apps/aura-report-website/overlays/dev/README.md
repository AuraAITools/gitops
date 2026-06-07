# Manifest Hydration

To hydrate the manifests in this repository, run the following commands:

```shell
git clone https://github.com/AuraAITools/gitops.git
# cd into the cloned directory
git checkout 9bbcc439474f28d6f4f385f85d9337d5ceaaf837
kustomize build ./apps/aura-report-website/overlays/dev
```
