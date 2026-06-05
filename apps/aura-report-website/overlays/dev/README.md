# Manifest Hydration

To hydrate the manifests in this repository, run the following commands:

```shell
git clone https://github.com/AuraAITools/gitops.git
# cd into the cloned directory
git checkout 1a47599ac49ffd49aa0814d7ec713229d5ff12b9
kustomize build ./apps/aura-report-website/overlays/dev
```
