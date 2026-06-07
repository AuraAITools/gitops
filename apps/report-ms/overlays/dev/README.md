# Manifest Hydration

To hydrate the manifests in this repository, run the following commands:

```shell
git clone https://github.com/AuraAITools/gitops.git
# cd into the cloned directory
git checkout 3afe8e0ccd1b0d81dd86a95eee932dc639916896
kustomize build ./apps/report-ms/overlays/dev
```
