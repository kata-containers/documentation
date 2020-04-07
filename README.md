# Documentation

* [Getting Started](#getting-started)
* [More User Guides](#more-user-guides)
* [Kata Container Use-Cases](#kata-container-use-cases)
* [Developer Guide](#developer-guide)
    * [Design and Implementations](#design-and-implementations)
    * [How to Contribute](#how-to-contribute)
    * [Code Licensing](#code-licensing)
    * [The Release Process](#the-release-process)
* [Help Improving the Documents](#help-improving-the-documents)
* [Website Changes](#website-changes)

The [Kata Containers](https://github.com/kata-containers)
documentation repository hosts overall system documentation, with information
common to multiple components.

For details of the other Kata Containers repositories, see the
[repository summary](https://github.com/kata-containers/kata-containers).

## Getting Started

* [Installation guides](./install/README.md): 
Install and run Kata Containers with Docker or Kubernetes

## More User Guides

* [Upgrading](Upgrading.md): How to upgrade from [Intel Clear Containers](https://github.com/clearcontainers) and [runV](https://github.com/hyperhq/runv) to [Kata Containers](https://github.com/kata-containers) and how to upgrade an existing Kata Containers system to the latest version.
* [Limitations](Limitations.md): Differences and limitations compared with the default [Docker](https://www.docker.com/) runtime,
[`runc`](https://github.com/opencontainers/runc).

### Howto guides

See the [how-to documentation](how-to).

## Kata Container Use-Cases

* [GPU Passthrough with Kata Containers](./use-cases/GPU-passthrough-and-Kata.md): Kata Containers supports passing certain GPUs from the host into the container.
* [OpenStack Zun with Kata Containers](./use-cases/zun_kata.md): How to get Kata Containers to work with OpenStack Zun using DevStack on Ubuntu 16.04.
* [SR-IOV with Kata Containers](./use-cases/using-SRIOV-and-kata.md): Setup to use SR-IOV with Kata Containers and Docker.
* [Intel QAT with Kata Containers](./use-cases/using-Intel-QAT-and-kata.md): Setup instructions to download kernel source, compile kernel and driver modules, load the kernel, and prepare a specially-built Kata Containers kernel and rootfs.
* [VPP with Kata Containers](./use-cases/using-vpp-and-kata.md): How to install and configure Vector Packet Processing (VPP) to improve router and switch functionality.
* [SPDK vhost-user with Kata Containers](./use-cases/using-SPDK-vhostuser-and-kata.md): Setup and run Storage Performance Development Kit (SPDK) vhost-user devices with Kata Containers and Docker.

## Developer Guide

Documents that help to understand and contribute to Kata Containers.

### Design and Implementations

* [Kata Containers architecture](design/architecture.md): Architectural overview of Kata Containers
* [Kata Containers design](./design/README.md): More Kata Containers design documents
* [Kata Containers threat model](./design/threat-model/threat-model.md): Kata Containers threat model

### How to Contribute

* [Developer Guide](Developer-Guide.md): Setup the Kata Containers development environments
* [How to contribute to Kata Containers](https://github.com/kata-containers/community/blob/master/CONTRIBUTING.md)
* [Code of Conduct](CODE_OF_CONDUCT.md)

### Code Licensing

* [Licensing](Licensing-strategy.md): About the licensing strategy of Kata Containers.

### The Release Process

* [Release strategy](Stable-Branch-Strategy.md)
* [Release process](Release-Process.md)

## Help Improving the Documents

* [Documentation requirements](Documentation-Requirements.md)

## Website Changes

If you have a suggestion for how we can improve the
[website](https://katacontainers.io), please raise an issue (or a PR) on
[the repository that holds the source for the website](https://github.com/OpenStackweb/kata-netlify-refresh).




<!-- This is a comment -->