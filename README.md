# kustomization-base

This repo contains a base kustomization that can be used to install Spinnaker to
a Kubernetes cluster. Please see the
[RFC](https://github.com/spinnaker/governance/blob/master/rfc/halyard-lite.md)
for an overview of the new Kubernetes-native installation pathway that is under
development.

This repo is under active development and is not ready for production (or even
development) use, but stay tuned for updates!

## Installing Spinnaker

### Prerequisites

Before you start, you will need:

- A Kubernetes cluster and `kubectl` configured to communicate with that cluster
- The [kustomize](https://github.com/kubernetes-sigs/kustomize/releases/latest)
  binary installed on your machine. Note that the version of _kustomize_ bundled
  with `kubectl` is out of date and will not work with these kustomizations.
- The [kleat](https://github.com/spinnaker/kleat) binary installed on your
  machine

### Instructions

Fork the [spinnaker-config](https://github.com/ezimanyi/spinnaker-config)
repository, and clone this fork to your local machine. This repository contains
a template kustomization that you will configure to deploy Spinnaker.

Change to the `base` directory; all further commands will assume that you are in
this directory.

At this point, you can generate the YAML needed to deploy Spinnaker by running
`kustomize build .`. As we have not configured any of the services, many would
fail on startup (particularly those that depend on some configured persistence
store), so let's add some configuration.

#### Create a `spinnaker` namespace

Create a new namespace called `spinnaker` into which each microservice will be
deployed:

```
kubectl create namespace spinnaker
```

If you would like to use a different namespace, you will need to update the
namespace in your root `kustomization.yml` and each microservice's `baseUrl` in
a fork of this repository's `service-discovery/spinnaker.yml`.

#### Configure Redis

A number of Spinnaker services require a Redis connection; this install pathway
requires there to be a service `redis` in the `spinnaker` namespace.

In the most common case, you will have an external redis; the template repo has
a `Service` configured with an `ExternalName`; you can update this to point to
the DNS name of your redis service.

For more complex cases, please refer to the following
[blog post](https://cloud.google.com/blog/products/gcp/kubernetes-best-practices-mapping-external-services)
on best practices for mapping external services. In general, the only
requirement of your solution is that you have a service named `redis` in the
`spinnaker` namespace that routes to a valid `redis`
backend.

Regardless of the approach you choose, add all the relevant redis Kubernetes
objects to your customization via `kustomize edit add resource redis/*.yml`.

#### Add any secret files

The `secrets` folder is intended to hold any secret files that are referenced by
your config files. The files in this folder will be mounted in `/var/secrets` in
each pod.

To add a secret, put it in the `secrets/` folder and add it to the `files` field
of the `spinnaker-secrets` entry in the `kustomization.yaml`. In this example,
we'll add a _kubeconfig_ file called `k8s-kubeconfig`:

```shell script
cp <path to kubeconfig> secrets/k8s-kubeconfig
```

and update the secret in the `kustomization.yaml` file as:

```yaml
- behavior: merge
  name: spinnaker-secrets
  files:
    - secrets/k8s-kubeconfig
```

#### Generate Spinnaker config files

This installation pathway expect the config files for your services to be in the
`kleat` directory. As the name suggests, the expectation is that these files
will be generated by `kleat` and written to this directory.

`kleat` takes as input a
[halconfig](https://github.com/spinnaker/kleat/blob/master/docs/docs.md). This
file is in general compatible with the file consumed by Halyard; if you are
migrating from Halyard, please read the minor changes you will need to make to
your file in the [kleat readme](https://github.com/spinnaker/kleat).

Unlike Halyard, `kleat` allows you to specify only the fields that you'd like to
configure. In this case, we will assume a very simple config:

```yaml
providers:
  kubernetes:
    enabled: true
    accounts:
      - name: k8s
        kubeconfigFile: /var/secrets/k8s-kubeconfig
    primaryAccount: k8s
```

Note that the `kubeconfigFile` argument here points to a file relative to
`/var/secrets`. This is important, as the `kleat` config requires that all file
references be relative to where they will be mounted in the running container
(rather than where they happen to be on the machine running kleat).

Save this file as `halconfig.yml` to the current directory, Invoke `kleat` to
consume the input `halconfig.yml` and output the service configs to the `kleat/`
directory:

```shell script
kleat halconfig.yml kleat/
```

This will generate service configs in the `kleat/` directory, replacing the
empty configs that were there before. The `kustomization.yaml` file is already
configured to mount these configs in the correct location in each microservice.
For example, it has an entry for the clouddriver config as:

```yaml
- behavior: merge
  files:
    - kleat/clouddriver.yml
  name: clouddriver-config
```
#### Enable optional services

The two optional Spinnaker services this workflow currently supports are Fiat
and Kayenta.

To enable Kayenta, ensure that `canary.enabled` is set to `true` in your hal
config, and then ensure the Kayenta kustomization is included in the `resources`
block of your `kustomization.yml`:

```
resources:
- github.com/spinnaker/kustomization-base/core
- github.com/spinnaker/kustomization-base/kayenta
```

To enable Fiat, ensure that `security.authz` is set to `true` in your hal
config, and then ensure the Fiat kustomization is included in the `resources`
block of your `kustomization.yml`:

```
resources:
- github.com/spinnaker/kustomization-base/core
- github.com/spinnaker/kustomization-base/fiat
```

#### (Optional) Add any -local configs

In addition to the main `service.yml` config file, each microservice also reads
in the contents of `service-local.yml` to support settings that are not
configurable in the _halconfig_.

If you would like to add a `-local.yml` config file for any service, add it to
the `local/` directory, and update that service's config in the
`kustomization.yaml` to also mount that `local.yml` file.

For example, to configure for clouddriver, add these settings to
`local/clouddriver-local.yml`, and update the `clouddriver-config` entry in the
`kustomization.yaml` to:

```yaml
- behavior: merge
  files:
    - kleat/clouddriver.yml
    - local/clouddriver-local.yml
  name: clouddriver-config
```

#### Deploy Spinnaker

Now that all of the config files are in place, you can generate the YAML files
to install Spinnaker by running `kustomize build .`. You can either save this to
a file and apply it, or directly pipe it to `kubectl` via:

```shell script
kustomize build . | kubectl apply -f -
```
