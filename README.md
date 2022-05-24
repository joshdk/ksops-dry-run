[![License][license-badge]][license-link]

# KSOPS Dry Run

🔓 Kustomize plugin to fake the decryption of ksops secrets

## Motivations

The [ksops](https://github.com/viaduct-ai/kustomize-sops) plugin is fantastic for managing encrypted secret resources as part of a Kustomize application.
One specific limitation is the inability to run `kustomize build` against any application that contains secrets that you do not have access to.
Since you do not have sufficient access to decrypt those secrets, the ksops plugin fails, and by extension so does the entire `kustomize build` invocation.

This comes up commonly in cases such as:
- A developer not being able to validate changes to their application in a production configuration.
- A CI pipeline not being able to validate every application across every environment.

## How it works

This repo provides a kustomize plugin that solves the above problems.

This plugin operates by subverting the job of the original `ksops` plugin.
It is intended that you rename the original `ksops` plugin from `${XDG_CONFIG_HOME}/kustomize/plugin/viaduct.ai/v1/ksops/ksops` to `${XDG_CONFIG_HOME}/kustomize/plugin/viaduct.ai/v1/ksops/_ksops` (notice the leading underscore in `_ksops`) and then install the `ksops-dry-run` plugin into the (now vacant) path of the original `ksops` plugin.

By default, when invoked this plugin will immediately exec the original `_ksops` plugin.
But if instead the variable `KSOPS_DRY_RUN` exists in the current working environment, then this plugin will perform its own custom functionality.

In this case, it acts identically to the original `ksops` plugin, but instead of producing decrypted secret resources, it instead produces secret resources where the (formerly encrypted) values are replaced with a placeholder value.   
This way you can run `kustomize build` and produce resource manifests for your application without actually needing to decrypt them. 

## Installation

To install, we need to rename the original `ksops` plugin to `_ksops`, download the `ksops-dry-run` plugin, and then rename `ksops-dry-run` to take the place of the original `ksops` plugin.

```shell
$ cd ${XDG_CONFIG_HOME}/kustomize/plugin/viaduct.ai/v1/ksops/
$ mv ksops _ksops
$ wget https://github.com/joshdk/ksops-dry-run/releases/download/v0.1.0/ksops-dry-run-amd64.tar.gz
$ tar -xf ksops-dry-run-amd64.tar.gz
$ cp ksops-dry-run ksops
```

### Uninstallation

To uninstall, we need to delete the `ksops-dry-run` plugin (which is currently named `ksops`), and finally restore the original `ksops` plugin. 

```shell
$ cd ${XDG_CONFIG_HOME}/kustomize/plugin/viaduct.ai/v1/ksops/
$ rm ksops
$ mv _ksops ksops
```

## Usage

Take for example, a simple kustomize app containing a single ksops encrypted secret:

```shell
$ ls

kustomization.yaml
secret-generator.yaml   
secret.enc.yaml
```

We can view that `secret.enc.yaml` file has been encrypted.

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: example-secret

stringData:
  SECRET_TOKEN: ENC[AES256_GCM,data:...type:str]

sops:
  kms:
    ...
```

If we run `kustomize build --enable-alpha-plugins .`, we can see that our secret is decrypted normally (potentially requiring a gpg key or other KMS API credentials):

```yaml
---
apiVersion: v1
kind: Secret
metadata:
  name: example-secret
stringData:
  SECRET_TOKEN: s00per_s3cret_t0k3n
```

But if instead we run `KSOPS_DRY_RUN= kustomize build --enable-alpha-plugins .`, we can see that our secret has been stubbed out with placeholder values, all without performing any actual decryption:

```yaml
---
apiVersion: v1
kind: Secret
metadata:
  name: example-secret
stringData:
  SECRET_TOKEN: KSOPS_DRY_RUN_PLACEHOLDER
```

## License

This code is distributed under the [MIT License][license-link], see [LICENSE.txt][license-file] for more information.

[license-badge]:         https://img.shields.io/badge/license-MIT-green.svg
[license-file]:          https://github.com/joshdk/ksops-dry-run/blob/master/LICENSE.txt
[license-link]:          https://opensource.org/licenses/MIT
