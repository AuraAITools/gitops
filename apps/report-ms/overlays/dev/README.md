# Manifest Hydration

To hydrate the manifests in this repository, run the following commands:

```shell
git clone https://github.com/AuraAITools/gitops.git
# cd into the cloned directory
git checkout 6b23052d747d6c4ac3e5bde806ba87bc41ebcfd5
kustomize build ./apps/report-ms/overlays/dev
```
