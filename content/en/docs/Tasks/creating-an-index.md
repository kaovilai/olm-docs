---
title: "Creating an Index of operator bundles"
weight: 3
description: >
  Add/Remove a collection of bundles to/from an Index
---

## Prerequisites

- [opm](https://github.com/operator-framework/operator-registry/releases) `v1.19.0+`

## Creating an Index

`OLM`'s `CatalogSource` [CRD][catalogsource-crd] accepts a container image reference to a catalog of operators that can
be made available to install in a cluster. You can make your operator bundle available to install in a cluster by adding
it to a catalog, packaging the catalog in a container image, and then using that image reference in the `CatalogSource`.
This image contains all of the metadata required for OLM to manage the lifecycles of all of the operators it contains.

OLM uses a plaintext [file-based catalog][file-based-catalog-spec] format (JSON or YAML) to store these records in an index, and `opm` has tooling
that helps initialize an index, render new records into it, and then validate that the index is valid. Let's walk
through a simple example.

First, we need to initialize our index, so we'll make a directory for it, generate a Dockerfile that can build an index
image, and then populate our index with our package definition.


### Initialize the index
```sh
$ mkdir example-operator-index
$ opm alpha generate dockerfile example-operator-index
$ opm init example-operator \
    --default-channel=preview \
    --description=./README.md \
    --icon=./example-operator.svg \
    --output yaml > example-operator-index/index.yaml
```

Let's validate our index to see if we're ready to ship!
```sh
$ opm validate example-operator-index
FATA[0000] invalid index:
└── invalid package "example-operator":
    └── invalid channel "preview":
        └── channel must contain at least one bundle
```

Alright, so it's not valid. It looks like we need to add a bundle, so let's do
that next...

### Add a bundle to the index

```sh
$ opm render quay.io/example-inc/example-operator-bundle:v0.1.0 \
    --output=yaml >> example-operator-index/index.yaml
```

Let's validate again:
```
$ opm validate example-operator-index
FATA[0000] package "example-operator", bundle "example-operator.v0.1.0" not found in any channel entries
```

### Add a channel entry for the bundle

We rendered the bundle, but we still haven't yet added it to any channels.
Let's initialize a channel:
```sh
cat << EOF >> example-operator-index/index.yaml
---
schema: olm.channel
package: example-operator
name: preview
entries:
  - name: example-operator.v0.1.0
EOF
```

Is the third time the charm for `opm validate`?

```sh
$ opm validate example-operator-index
$ echo $?
0
```

Success! There were no errors and we got a `0` error code.

In the general case, adding a bundle involves three discreet steps:
- Render the bundle into the index using `opm render <bundleImage>`.
- Add the bundle into desired channels and update the channels' upgrade edges
  to stitch the bundle into the correct place.
- Validate the resulting index.

> NOTE: Index metadata should be stored in a version control system (e.g. `git`) and index images should be rebuilt from source
whenever updates are made to ensure that all index changes are auditable.

**Step 1** is just a simple `opm render` command.

**Step 2** has no defined standards other than that the result must pass validation in step 3. Some operator authors may
decide to hand edit channels and upgrade edges. Others may decide to implement automation (e.g. to idempotently
build semver-based channels and upgrade graphs based solely on the versions of the operators in the package). There is
no right or wrong answer for implementing this step as long as `opm validate` is successful.

There are some guidelines to keep in mind though:
- Once a bundle is present in an index, you should assume that one of your users has installed it. With that in mind,
  you should take care to avoid stranding users that have that version installed. Put another way, make sure that
  all previously published bundles in an index have a path to the current/new channel head.
- Keep the semantics of the upgrade edges you use in mind. `opm validate` is not able to tell you if you have a sane
  upgrade graph. To learn more about the upgrade graph of an operator, checkout the
  [creating an upgrade graph doc][upgrade-graph-doc].

### Build and push the index image

Lastly, we can build and push our index:

```sh
$ docker build . \
    -f example-operator-index.Dockerfile \
    -t quay.io/example-inc/example-operator-index:latest
$ docker push quay.io/example-inc/example-operator-index:latest
```

Now the index image is available for clusters to use and reference with `CatalogSources` on their cluster.

[catalogsource-crd]: /docs/concepts/crds/catalogsource
[file-based-catalog-spec]: /docs/reference/file-based-catalogs
[upgrade-graph-doc]: /docs/concepts/olm-architecture/operator-catalog/creating-an-update-graph
