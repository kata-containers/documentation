# How to use rootless Kata Containers with Podman

* [How to use rootless Kata Containers with Podman](#how-to-use-rootless-kata-containers-with-podman)
    * [Requirements](#requirements)
    * [Installation](#installation)
    * [Configuration](#configuration)
            * [1. Disable SELinux](#1-disable-selinux)
            * [2. Add user to KVM group](#2-add-user-to-kvm-group)
            * [3. Reboot](#3-reboot)
            * [5. Disable `vhost-net`](#5-disable-vhost-net)
            * [6. Modify the Kata images permissions](#6-modify-the-kata-images-permissions)
            * [7. Set up Podman rootless configuration](#7-set-up-podman-rootless-configuration)
            * [8. Add Kata Runtime to Podman configuration file (optional)](#8-add-kata-runtime-to-podman-configuration-file-optional)
            * [9. Set Kata runtime as Podman's default OCI runtime (optional)](#9-set-kata-runtime-as-podmans-default-oci-runtime-optional)
    * [Run Kata with rootless Podman](#run-kata-with-rootless-podman)
    * [Appendix: Possible Errors](#appendix-possible-errors)
        * [Error caused by agent or runtime version mismatch](#error-caused-by-agent-or-runtime-version-mismatch)
        * [Missing registry file](#missing-registry-file)


For an even more secure system, [Kata Containers](https://Katacontainers.io)
can run workloads without a privileged user. Using
[Podman](https://podman.io/) as the container engine, and
[`slirp4netns`](https://github.com/rootless-containers/slirp4netns) for
user-space networking.

## Requirements

- A Linux system, see
  [supported distributions](https://github.com/kata-containers/documentation/blob/master/install/README.md#supported-distributions)
  for an updated list.

  - If using CentOS 7, `newuidmap` and `newgidmap` do not exist, and can be installed with:

    ```bash
    $ (git clone https://github.com/shadow-maint/shadow; cd shadow; ./autogen.sh --prefix=/usr --enable-man; make && sudo make -C src install)
    $ sudo bash -c 'echo 10000 > /proc/sys/user/max_user_namespaces'
    $ sudo bash -c "echo $(whoami):110000:65536 > /etc/subuid"
    $ sudo bash -c "echo $(whoami):110000:65536 > /etc/subgid"
    ```

- Golang 1.12

## Installation

Refer to the following table for the __minimum__ required software versions
and the installation instructions:

| Component       | Version | Install Instructions|
| ----------------|:-------:|---------------------|
| Podman          | WIP     | [see here](https://github.com/containers/libpod/blob/master/install.md)
| `slirp4netns`   | 0.4.0   | [see here](https://github.com/rootless-containers/slirp4netns#quick-start)
| Kata Containers | WIP     | [see here](https://github.com/kata-containers/documentation/blob/master/install/README.md)

> **NOTE:**
>
> If installing Podman with a package manager, there is usually no need to
> install `slirp4netns` separately.

## Configuration

Now that Kata Containers and Podman have been installed, they need to be
configured for rootless execution.

### Disable SELinux

If SELinux is installed and enabled, it needs to be disabled with the
following command (Kata Containers
[does not support SELinux](https://github.com/kata-containers/documentation/blob/master/Limitations.md#selinux-support)).

> **Warning:**
> The following command may differ depending on the distro being used:

```bash
$ [ -f /etc/selinux/config ] && sudo sed -i 's/^SELINUX=.*/SELINUX=disabled/g' /etc/selinux/config
```

### Add user to KVM group

If running a KVM based hypervisor, the user running the workload needs to be added to the KVM group:

```bash
$ sudo usermod -a -G kvm $USER
```

### Reboot

Reboot the system for the changes to take effect (a reboot is required when
disabling SELinux, while logging out and back in is enough to have that user
joining the `KVM` group).

You can now verify if the configuration is correct:

* (if installed) SELinux should have been disabled:
```bash
$ getenforce
Disabled
```

* The user should be in the `kvm` group:
```
$ groups | grep -ow kvm
kvm
```

### Setup Kata configuration files

If the Kata `configuration.toml` file does not exist in `/etc`, do the
following:

```bash
$ sudo install -D -o ${USER} -g root -m 0640 /usr/share/defaults/kata-containers/configuration.toml /etc/kata-containers
```

Or, if the file exists, but is not readable by the user:

```bash
$ sudo chown -R a+r /etc/kata-containers/
```

### Disable `vhost-net`

Disable `vhost-net` in the Kata configuration file, by commenting out the
`disable_vhost_net` line:

```bash
$ sudo sed -i -e 's/^#disable_vhost_net = true/disable_vhost_net = true/' /etc/kata-containers/configuration.toml
```

### Modify the Kata images permissions

The Kata images needs to be accessible by the user, so change the permissions
of the image directory to be readable by the user.

```bash
$ sudo chown -R a+r /usr/share/kata-containers
```

### Set up Podman rootless configuration

If `libpod.conf` does not exist in `~/.config/containers/`:

```bash
$ sudo install -D -o ${USER} -g ${USER} -m 0640 /usr/share/containers/libpod.conf ~/.config/containers/
```

By default the `tmp_dir` in `libpod.conf` is set to `/var/run/libpod`, however
that is not accessible by a user, so change to the rootless runtime directory.

```bash
$ sed -i -e "s|^tmp_dir = .*$|tmp_dir = \"$XDG_RUNTIME_DIR/libpod/tmp\"|" ~/.config/containers/libpod.conf
```

### Add Kata Runtime to Podman configuration file (optional)

You can tell Podman to create or run containers using the Kata runtime with
`--runtime=value` flag.

You can directly pass the fully qualified Kata runtime path with:

```bash
$ podman run --runtime=/usr/local/bin/kata-runtime ...
```

or add a `kata` entry in the `[runtimes]` section of the configuration file by
appending the Kata Runtime binary path(s) to the `libpod.conf` file:

```bash
$ echo 'kata = ["/usr/local/bin/kata-runtime"]' >> ~/.config/containers/libpod.conf
```

and then use `kata` as the runtime name:

```bash
$ podman run --runtime=kata ...
```

> **NOTE:**
>
> A less recommended approach could be to have the absolute `kata-runtime`
> path in the standard `$PATH` location instead of the configuration file, and
> a binary with that name will be looked up automatically:
>
> ```bash
> kata-runtime = [
> ]
> ```

### Set Kata runtime as Podman's default OCI runtime (optional)

To avoid using the `--runtime=value` flag set Kata runtime as the default
runtime:

```bash
$ sed -i -e 's/^runtime = "runc"/runtime = "kata"/' ~/.config/containers/libpod.conf
```

## Run Kata with rootless Podman

```bash
$ podman run --rm --runtime=kata alpine date
```

> **NOTE:**
>
> To obtain debug logs you can:
>
>  - Enable
>    [debug](https://github.com/kata-containers/documentation/blob/master/Developer-Guide.md#enable-full-debug)
>    in Kata (logs are added to journald).
>
>  - Pass `--log-level=debug` to Podman (logs are printed to stderr).

## Appendix: Possible Errors

If you are building from source you may encounter the following errors.

### Error caused by agent or runtime version mismatch

```
rpc error: code = Internal desc = Could not add route dest()/gw(10.0.2.2)/dev(tap0): network is unreachable: OCI runtime error
```

**Solution:**

You may need to
[rebuild the agent](https://github.com/kata-containers/documentation/blob/master/Developer-Guide.md#add-a-custom-agent-to-the-image---optional);
there was a change in both the
[agent](https://github.com/kata-containers/agent/commit/a78e8cfda627cc350dc9d9ca9b969ebb642030c3)
and
[runtime](https://github.com/kata-containers/runtime/commit/cfedb06a19135e2ab4f18203a4f3147cdc3a4980)
code. This would probably only occur if building latest from source, since the
runtime version would have the change and the released agent does not.

### Missing registry file

```
Error: unable to pull alpine: image name provided is a short name and no search registries are defined in the registries config file.
```

**Solution:**

The `/etc/containers/registries.conf` file is missing; either use the full
image name, or add the [configuration
file](https://github.com/containers/libpod/blob/master/install.md#configuration-files).
