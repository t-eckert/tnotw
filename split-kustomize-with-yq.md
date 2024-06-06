# Split up Kustomize Output with yq

[Kustomize](https://kustomize.io/) is an incredibly powerful tool for integrating
existing Kubernetes manifests in your own cluster. You can write a configuration
file in YAML that references the manifest you want to install. That manifest can
include local files as well as remote repositories.

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - deployment.yaml
  - github.com/kubernetes-sigs/gateway-api/config/crd/experimental?ref=v0.6.2
```

This configuration will pull in the local `deployment.yaml` as well as the
`CustomResourceDefinitions` that are hosted in the Gateway API repository.

When you run `kustomize build` the combined YAML to configure your cluster
will be sent to `stdout`. This can be piped into a `.yaml` file if you
would like to examine or save the configuration.

However, Kustomize will only output the config as a single YAML file with
multiple "documents" concatenated together. If you want to split this output
across multiple files, the [`yq`](https://mikefarah.gitbook.io/yq/) command line
utility provides a handy way of
[doing just that](https://mikefarah.gitbook.io/yq/usage/split-into-multiple-files).

```bash
kustomize build | yq --split-exp '.metadata.name + ".yaml"' --no-doc
```

Here, we pipe the output of `kustomize build` to `yq`. Passing the `--split-exp` flag
tells `yq` to split the input into separate documents. It takes an argument that
uses the `yq` expression language (which is the same as
[`jq`](https://stedolan.github.io/jq/)) to name the files based on the value in the
YAML document's `metadata.name` field. The `--no-doc` flag simply omits the `---` used
to separate YAML documents from the output.
