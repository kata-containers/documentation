# How to use rootless Kata Containers with Podman

* [How to use rootless Kata Containers with Podman](#how-to-use-rootless-kata-containers-with-podman)
    * [Requirements](#requirements)
    * [Installation](#installation)
    * [Configuration](#configuration)
        * [Disable SELinux](#disable-selinux)
        * [Add user to KVM group](#add-user-to-kvm-group)
        * [Reboot](#reboot)
        * [Setup Kata configuration files](#setup-kata-configuration-files)
        * [Disable `vhost-net`](#disable-vhost-net)
        * [Modify the Kata image permissions](#modify-the-kata-image-permissions)
        * [Set up Podman rootless configuration](#set-up-podman-rootless-configuration)
        * [Add Kata Runtime to Podman configuration file (optional)](#add-kata-runtime-to-podman-configuration-file-optional)
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

- A Linux system

  See
  [supported distributions](https://github.com/kata-containers/documentation/blob/master/install/README.md#supported-distributions)
  for an updated list.

  - If using an older distribution such as CentOS 7, `newuidmap` and
    `newgidmap` will not exist. Install them with:

    ```bash
    $ if [ ! -f /etc/subuid ]; then
    $   (git clone https://github.com/shadow-maint/shadow && cd shadow && ./autogen.sh --prefix=/usr && make && sudo make -C src install)
    $   sudo bash -c 'echo 10000 > /proc/sys/user/max_user_namespaces'
    $   sudo bash -c "echo $(whoami):110000:65536 > /etc/subuid"
    $   sudo bash -c "echo $(whoami):110000:65536 > /etc/subgid"
    $ fi
    ```

- Golang version 1.12 or newer.

## Installation

Refer to the following table for the *minimum* required software versions and
the installation instructions:

| Component       | Version       | Install Instructions|
| ----------------|:-------------:|---------------------|
| Podman          | 1.6.2         | [see here](https://github.com/containers/libpod/blob/master/install.md) |
| `slirp4netns`   | 0.4.0         | [see here](https://github.com/rootless-containers/slirp4netns#quick-start) |
| Kata Containers | 1.10.0-alpha1 | [see here](https://github.com/kata-containers/documentation/blob/master/install/README.md) |
| Host Kernel     | 4.14          | |

Rootless support for Kata has been verified with the QEMU hypervisor. It is
recommended to use the QEMU binary from the Kata packages for running rootless
containers.

> **NOTE:**
>
> If installing Podman with a package manager, there is usually no need to
> install `slirp4netns` separately.
> Rootless support for Kata is available in Kata 1.10.0-alpha1 release.

## Configuration

Now that you have installed Kata Containers and Podman, you need to configure
them for rootless execution.

### Disable SELinux

> **Warning:**
>
> You should understand the security implications before disabling SELinux.

If SELinux is installed and enabled, it needs to be disabled with the
following command since Kata Containers
[does not support SELinux](https://github.com/kata-containers/documentation/blob/master/Limitations.md#selinux-support).

> **Warning:**
> The following command might differ depending on the distribution you use:

```bash
$ [ -f /etc/selinux/config ] && sudo sed -i 's/^SELINUX=.*/SELINUX=disabled/g' /etc/selinux/config
```

### Add user to KVM group

- To enable rootless support, the user running the workload needs to be added
  to the `kvm` group (which may need to be created as shown):

  ```bash
  $ getent group kvm &>/dev/null || sudo groupadd --system kvm
  $ sudo usermod -a -G kvm $USER
  ```

  > **NOTE:**
  >
  > The `kvm` group owns the `/dev/kvm` device on most distributions.

- Ensure the `/dev/kvm` device is group owned, readable and writable by the `kvm` group:

  ```bash
  $ sudo chown root:kvm /dev/kvm
  $ sudo chmod g+rw /dev/kvm
  ```

### Reboot

Reboot the system for the changes to take effect (when you disable SELinux you
must reboot, while logging out and back in is enough to have that user joining
the `kvm` group).

Verify the configuration is correct:

- Host kernel minimum version requirement is satisfied

  ```bash
  $ host_kernel_version=$(uname -r|cut -d. -f1-2)
  $ result=$(echo "$host_kernel_version >= 4.14"|bc)
  $ [ "$result" -ne 1 ] && echo "ERROR: host kernel version too old" && exit 1
  ```

- If installed, ensure SELinux is disabled:

  ```bash
  $ [ -n "$(command -v getenforce)" ] && [ $(getenforce) != Disabled ] && echo "ERROR: SELinux must be disabled" && exit 1
  ```

- The user should be in the `kvm` group:

  ```bash
  $ [ -z $(groups | grep -ow kvm) ] && echo "ERROR: user is not a member of the kvm group" && exit 1
  ```

### Setup Kata configuration files

This set of instructions are based on the assumption that Kata Containers has
been installed using a package manager. If `kata-deploy` was used, the
binaries and `configuration.toml` files will be located at `/opt/kata/` and
the following instructions will have to be modified accordingly.

If the Kata `configuration.toml` file does not exist in `/etc`, create it and
ensure the file is readable by all:

```bash
$ cfg="/etc/kata-containers/configuration.toml"
$ sudo mkdir -p $(dirname "$cfg")
$ [ ! -e "$cfg" ] && sudo install -D /usr/share/defaults/kata-containers/configuration.toml $(dirname "$cfg")
$ sudo chown root:kvm "$cfg"
$ sudo chmod g+r "$cfg"
```

### Disable `vhost-net`

This step will not be required in case you are using Kata based on the latest source.

Disable `vhost-net` in the Kata configuration file by commenting out the
`disable_vhost_net` line:

```bash
$ sudo sed -i -e 's/^#disable_vhost_net = true/disable_vhost_net = true/' /etc/kata-containers/configuration.toml
```

The above step is required in Kata version `1.10.0-alpha1`. In future
releases, this should no longer be required as the Kata runtime will handle
this automatically for the rootless case. Disabling `vhost-net` will degrade
network performance slightly.

### Modify the Kata image permissions

Upstream QEMU needs to have read and write permissions for the Kata image in
order to use it as a NVDIMM memory backend, even though in case of Kata, the
image is used purely for a read-only operation.

Write access on the image should not be required for a read-only access. We
have fixed this in the QEMU packages that we ship with Kata and plan on
up-streaming this fix.

(Link to the patch: https://github.com/kata-containers/packaging/blob/master/qemu/patches/4.1.x/0002-memory-backend-file-nvdimm-support-read-only-files-a.patch)

In case you are using a QEMU binary provided by your distribution, it is
recommended that you use an initrd instead of rootfs image file. The reason
being, you will need to add write permissions to the rootfs image for the user
running the workload, allowing the user to modify the image. 

If you still wish to to use the rootfs image instead of initrd, you can change
the group ownership of the image by choosing a trusted group and adding read
and write (`rw`) permissions for that group. Choose a group where members of
the group trust each other, as giving write access to the group allows any
member of the group to modify the image.

```bash
$ TRUSTED_GROUP=kvm
$ img=$(readlink -f /usr/share/kata-containers/kata-containers.img)
$ sudo chown -R "root:$TRUSTED_GROUP" "$img"
$ sudo chmod -R g+rw "$img"
```
> **Warning:**
>
> You should understand the security implications of the above step before
> adding write permissions for the image.

### Set up Podman rootless configuration

If `libpod.conf` does not exist in `~/.config/containers/`:

```bash
$ [ -e ~/.config/containers/libpod.conf ] || install -D /usr/share/containers/libpod.conf ~/.config/containers/libpod.conf
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
$ podman run --runtime=/usr/bin/kata-runtime alpine date
```

Alternatively, add a `kata` entry in the `[runtimes]` section of the
configuration file by appending the Kata Runtime binary path(s) to the
`libpod.conf` file:

```bash
$ echo 'kata = ["/usr/bin/kata-runtime"]' >> ~/.config/containers/libpod.conf
```

You can then specify the runtime name as `kata`:

```
$ podman run --rm --runtime=kata alpine date
```

> **NOTE:**
>
> A less recommended approach could be to have the absolute `kata-runtime`
> path in the standard `$PATH` location instead of the configuration file. In
> this case it looks up a binary with that name automatically:
>
> ```
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
>    in Kata (this adds the logs to journald).
>
>  - Pass `--log-level=debug` to Podman (this prints the logs to stderr).

## Appendix: Possible Errors

If you are building from source you might encounter the following errors.

### Error caused by agent or runtime version mismatch

```
rpc error: code = Internal desc = Could not add route dest()/gw(10.0.2.2)/dev(tap0): network is unreachable: OCI runtime error
```

**Solution:**

You might need to
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
image name, or add the
[configuration file](https://github.com/containers/libpod/blob/master/install.md#configuration-files).
