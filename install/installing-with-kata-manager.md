# Installing with `kata-manager`

* [Introduction](#introduction)
* [Full Installation](#full-installation)
* [Install the Kata packages only](#install-the-kata-packages-only)
* [Further Information](#further-information)

## Introduction
`kata-manager` automates the Kata Containers installation procedure documented for [these Linux distributions](README.md#supported-distributions).

> **Note**:
> - `kata-manager` requires `curl` and `sudo` installed on your system.
>
> - Full installation mode is only available for Docker container manager. For other setups, you
> can still use `kata-manager` to [install Kata package](#install-the-kata-packages-only), and then setup your container manager manually.
>
> - You can run `kata-manager` in dry run mode by passing the `-n` flag. Dry run mode allows you to review the
> commands that `kata-manager` would run, without doing any change to your system.


## Full Installation
This command does the following:
1. Installs Kata Containers packages
2. Installs Docker
3. Configure Docker to use the Kata OCI runtime by default

```bash
$ bash -c "$(curl -fsSL https://raw.githubusercontent.com/kata-containers/tests/master/cmd/kata-manager/kata-manager.sh) install-docker-system"
```

<!--
You can ignore the content of this comment.
(test code run by test-install-docs.sh to validate code blocks this document)

```bash
$ bash -c "$(curl -fsSL https://raw.githubusercontent.com/kata-containers/tests/master/cmd/kata-manager/kata-manager.sh) remove-packages"
```
-->
## Install the Kata packages only
Use the following command to only install Kata Containers packages.

```bash
$ bash -c "$(curl -fsSL https://raw.githubusercontent.com/kata-containers/tests/master/cmd/kata-manager/kata-manager.sh) install-packages"
```

## Validate the installation
After kata-manager has been install, use the following command to validate the installation.

```bash
$ kata-runtime kata-check
```
If everything is properly installer, it should say *'System is capable of running Kata Containers'*.

You can additionally use this command to get all details about the installation

```bash
$ kata-runtime kata-env
```
You will obtain a detailled list of configuration values similar to the following example.

```
[Meta]
  Version = "1.0.24"

[Runtime]
  Debug = false
  Trace = false
  DisableGuestSeccomp = true
  DisableNewNetNs = false
  SandboxCgroupOnly = false
  Path = "/usr/bin/kata-runtime"
  [Runtime.Version]
    OCI = "1.0.1-dev"
    [Runtime.Version.Version]
      Semver = "1.12.0-alpha1"
      Major = 1
      Minor = 12
      Patch = 0
      Commit = ""
  [Runtime.Config]
    Path = "/usr/share/defaults/kata-containers/configuration.toml"

[Hypervisor]
  MachineType = "pc"
  Version = "QEMU emulator version 5.0.0\nCopyright (c) 2003-2020 Fabrice Bellard and the QEMU Project developers"
  Path = "/usr/bin/qemu-vanilla-system-x86_64"
  BlockDeviceDriver = "virtio-scsi"
  EntropySource = "/dev/urandom"
  SharedFS = "virtio-9p"
  VirtioFSDaemon = "/usr/bin/virtiofsd"
  Msize9p = 8192
  MemorySlots = 10
  PCIeRootPort = 0
  HotplugVFIOOnRootBus = false
  Debug = false
  UseVSock = false

[Image]
  Path = "/usr/share/kata-containers/kata-containers-image_clearlinux_1.12.0-alpha1_agent_8c9bbadcd4.img"

[Kernel]
  Path = "/usr/share/kata-containers/vmlinuz-5.4.60.88-50.container"
  Parameters = "systemd.unit=kata-containers.target systemd.mask=systemd-networkd.service systemd.mask=systemd-networkd.socket scsi_mod.scan=none"

[Initrd]
  Path = ""

[Proxy]
  Type = "kataProxy"
  Path = "/usr/libexec/kata-containers/kata-proxy"
  Debug = false
  [Proxy.Version]
    Semver = "1.12.0-alpha1-c604a60"
    Major = 1
    Minor = 12
    Patch = 0
    Commit = "<<unknown>>"

[Shim]
  Type = "kataShim"
  Path = "/usr/libexec/kata-containers/kata-shim"
  Debug = false
  [Shim.Version]
    Semver = "1.12.0-alpha1-d6df720"
    Major = 1
    Minor = 12
    Patch = 0
    Commit = "<<unknown>>"

[Agent]
  Type = "kata"
  Debug = false
  Trace = false
  TraceMode = ""
  TraceType = ""

[Host]
  Kernel = "5.4.0-1028-gcp"
  Architecture = "amd64"
  VMContainerCapable = true
  SupportVSocks = true
  [Host.Distro]
    Name = "Ubuntu"
    Version = "20.04"
  [Host.CPU]
    Vendor = "GenuineIntel"
    Model = "Intel(R) Xeon(R) CPU @ 2.20GHz"

[Netmon]
  Path = "/usr/libexec/kata-containers/kata-netmon"
  Debug = false
  Enable = false
  [Netmon.Version]
    Semver = "1.12.0-alpha1"
    Major = 1
    Minor = 12
    Patch = 0
    Commit = "<<unknown>>"
```

## Further Information
For more information on what `kata-manager` can do, refer to the [`kata-manager` page](https://github.com/kata-containers/tests/blob/master/cmd/kata-manager).
