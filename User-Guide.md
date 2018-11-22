# Kata Containers User Guide

** THIS DOCUMENT IS A WorkInProgress **

* [Kata Containers User Guide](#kata-containers-user-guide)
   * [What is Kata Containers?](#what-is-kata-containers)
* [Installation](#installation)
   * [Supported platforms](#supported-platforms)
   * [docker](#docker)
      * [compose](#compose)
   * [Kubernetes](#kubernetes)
      * [pods](#pods)
   * [Zun](#zun)
* [Configuration](#configuration)
   * [Memory](#memory)
      * [memory allocation](#memory-allocation)
   * [CPUs](#cpus)
   * [filesystems and storage](#filesystems-and-storage)
      * [host graph drivers](#host-graph-drivers)
         * [non-block exposed (overlay et. al.)](#non-block-exposed-overlay-et-al)
         * [block exposed (devicemapper et. al.)](#block-exposed-devicemapper-et-al)
      * [rootfs mounts](#rootfs-mounts)
      * [volume mounts](#volume-mounts)
      * [Types of filesystems](#types-of-filesystems)
      * [SPDK](#spdk)
      * [CEPH et. al.](#ceph-et-al)
   * [VM features](#vm-features)
      * [Hypervisors](#hypervisors)
      * [Kernels](#kernels)
         * [tracking stable](#tracking-stable)
         * [required features](#required-features)
         * [kernel-per-pod](#kernel-per-pod)
      * [rootfs](#rootfs)
         * [images](#images)
         * [initrd](#initrd)
      * [PC types](#pc-types)
      * [QEMU versions](#qemu-versions)
         * [NEMU](#nemu)
      * [Random and entropy](#random-and-entropy)
      * [direct hardware mapping](#direct-hardware-mapping)
         * [SR-IOV](#sr-iov)
         * [docker device arguments](#docker-device-arguments)
      * [notes on scaling](#notes-on-scaling)
      * [migration](#migration)
      * [KSM](#ksm)
      * [DAX](#dax)
      * [balloooning](#balloooning)
      * [pinning on the host](#pinning-on-the-host)
   * [networking](#networking)
      * [veth](#veth)
      * [macvtap](#macvtap)
      * [CNI](#cni)
      * [CNM](#cnm)
      * [DPDK](#dpdk)
      * [VPP](#vpp)
   * [Security layers](#security-layers)
      * [SElinux](#selinux)
      * [seccomp](#seccomp)
      * [AppArmor](#apparmor)
* [Appendix](#appendix)
   * [Things that are missing...](#things-that-are-missing)

This Kata Containers User Guide aims to be a comprehensive guide to the explanation of,
installation, configuration, and use of Kata Containers.

The Kata Containers [source repositories](https://github.com/kata-containers) contain
significant amounts of other documentation, covering some subjects in more detail.

## What is Kata Containers?

Kata Containers is, as defined in the [community repo](https://github.com/kata-containers/community):

> Kata Containers is an open source project and community working to build a standard implementation of lightweight Virtual Machines (VMs) that feel and perform like containers, but provide the workload isolation and security advantages of VMs.

It is a drop in additional OCI compatible container runtime, which can therefore be used
with [Docker] and [Kubernetes].

# Installation

## Supported platforms

Kata Containers is primarily a Linux based application. It can be installed on the most
common Linux distributions, using the common Linux packaging tools.


Details on installation can be found in [documentation repository](https://github.com/kata-containers/documentation/tree/master/install).

For the curious, adventurous, developers or those using a distribution not presently
supported with pre-built pacakges, Kata Containers can be
[installed from source](https://github.com/kata-containers/documentation/blob/master/Developer-Guide.md). If you are on a distribution that is not presently supported, please feel free
to reach out to the [community](https://github.com/kata-containers/community#community) to
discuss adding support. Of course, contributions are [most welcome](https://github.com/kata-containers/community/blob/master/CONTRIBUTING.md).

## docker

Kata Containers can be installed into Docker as an additional container runtime. This does
not remove any functionality from Docker. You can choose which container runtime is the
default for Docker if none is specified. You can run Kata Container runtime containers
in parallel with other contianers using a different container runtime (such as the default
Docker `runc` runtime).

Instructions on how to configure Docker to add Kata Containers as a runtime can be found
in the [documentation repository](https://github.com/kata-containers/documentation/tree/master/install/docker)

### compose

It should be noted, that presently Kata Containers may not function fully in all
`docker compose` situations. In particular, Docker compose makes use of network links to
supply its own internal DNS service, which is difficult for Kata Containers to replicate.
Work is on-going, and the Kata Containers [limitations](https://github.com/kata-containers/documentation/blob/master/Limitations.md)
document can be checked for the present state.

## Kubernetes

Kata Containers can be integrated as a runtime into Kubernetes. Kata Containers can be
integrated via either CRI-containerd or CRI-O.

For details on configuring Kata Containers with CRI-containerd see this [document](https://github.com/kata-containers/documentation/blob/master/how-to/how-to-use-k8s-with-cri-containerd-and-kata.md)

### pods

Note that pods have some different functionality from straight docker - and note them
throughout the document (such as memory and cpu scaling).

## Zun

Kata Containers can be used as a runtime for OpenStack by integrating with [Zun](https://wiki.openstack.org/wiki/Zun). Details on how to set this integration up can be found in [this document](https://github.com/kata-containers/documentation/blob/master/zun/zun_kata.md)

# Configuration

Kata Containers has a comprehensive TOML based configuration file. Much of the information
on the available configuration options is contained directly in [that file](https://github.com/kata-containers/runtime/blob/master/cli/config/configuration.toml.in).
This section expands on some of the details and finer points of the configuration.

## Memory

As Kata Containers runs containers inside VMs it differs from software containers in
how memory is allocated and restricted to the container. VMs are allocated an amount
of memory, whereas software containers can run either unconstrained (they can access,
and share with other containers, all of the host memory), or they can have some constraints
imposed upon them via [hard or soft limits](https://docs.docker.com/config/containers/resource_constraints/#limit-a-containers-access-to-memory).

### memory allocation

If no constraints are set, then Kata Containers will set the VM memory size using a
combination of the [value set in the runtime config file](https://github.com/kata-containers/runtime/blob/master/cli/config/configuration.toml.in#L80-L83), which is [2048 MiB](https://github.com/kata-containers/runtime/blob/master/Makefile#L131-L132) by default, plus 
the addition of the requested constraint.

Kata Containers gets the memory constraint information from the OCI JSON file passed to it
by the orchestration layer. In the case of Docker, these can be set on the [command line](https://docs.docker.com/config/containers/resource_constraints/#memory)
For Kubernetes, you can set up [memory limits and requests](https://kubernetes.io/docs/tasks/administer-cluster/manage-resources/memory-default-namespace/)

> **Note** We should detail how limits and requests map into Kata VMs.

## CPUs

If the container orchestrator provides CPU constraints, then Kata Containers configures the
VM per those constraints (rounded up to the nearest whole CPU), plus one extra CPU (to cater
for any VM overheads).

## filesystems and storage

### host graph drivers

Briefly explain that Kata maps in rootfs differently depending on the host side graph driver. block or 9p.

#### non-block exposed (overlay et. al.)

#### block exposed (devicemapper et. al.)

### rootfs mounts

### volume mounts

### Types of filesystems

### SPDK

### CEPH et. al.

## VM features

### Hypervisors

### Kernels

#### tracking stable

#### required features

#### kernel-per-pod

### rootfs

#### images

#### initrd

### PC types

### QEMU versions

#### NEMU

### Random and entropy

### direct hardware mapping

#### SR-IOV

#### docker device arguments

### notes on scaling
ptys, file handles, network size.

### migration

### KSM

### DAX

### balloooning

### pinning on the host

## networking

### veth

### macvtap

### CNI

### CNM

### DPDK

### VPP

## Security layers

### SElinux

On the host, and in the container.

### seccomp

### AppArmor

# Appendix

## Things that are missing...

entropy
