# How to use rootless Kata Containers with Podman

* [How to use rootless Kata Containers with Podman](#how-to-use-rootless-kata-containers-with-podman)
    * [Requirements](#requirements)
    * [Installation](#installation)
    * [Configuration](#configuration)
        * [Disable SELinux](#disable-selinux)
        * [Add user to KVM group](#add-user-to-kvm-group)
        * [Reboot](#reboot)
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

- A Linux system, see
  [supported distributions](https://github.com/kata-containers/documentation/blob/master/install/README.md#supported-distributions)
  for an updated list.

  - If using CentOS 7, `newuidmap` and `newgidmap` do not exist. Install them with:

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

| Component       | Version       | Install Instructions|
| ----------------|:-------------:|---------------------|
| Podman          | 1.6.2         | [see here](https://github.com/containers/libpod/blob/master/install.md)
| `slirp4netns`   | 0.4.0         | [see here](https://github.com/rootless-containers/slirp4netns#quick-start)
| Kata Containers | 1.10.0-alpha1 | [see here](https://github.com/kata-containers/documentation/blob/master/install/README.md)
| Host  Kernel    | 4.14          | 

Rootless support for Kata has been verified with `qemu` hypervisor.
It is recommended to use qemu binary from Kata packages for running rootless containers.

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
following command (Kata Containers
[does not support SELinux](https://github.com/kata-containers/documentation/blob/master/Limitations.md#selinux-support)).

> **Warning:**
> The following command might differ depending on the distro you use:

```bash
$ [ -f /etc/selinux/config ] && sudo sed -i 's/^SELINUX=.*/SELINUX=disabled/g' /etc/selinux/config
```

### Add user to KVM group

To enable rootless support, the user running the workload needs to be added to the `kvm` group.

```bash
$ sudo usermod -a -G kvm $USER
```
> **NOTE:**
>
> `kvm` should be the group owning the device node `/dev/kvm` on most distros.
> Make sure the minimum permissions on `/dev/kvm` are as shown below:
> ```
> $ ls -la /dev/kvm
> crw-rw---- 1 root kvm 10, 232 Nov 11 20:28 /dev/kvm
> ```
> If the group owner happens to be `root`, you may need to create a system group `kvm` and
> change permissions for `/dev/kvm` as shown above.
> ```
> $ sudo addgroup --system kvm
> $ sudo chown root:kvm /dev/kvm
> $ sudo chmod g+rw /dev/kvm
> ```
 
### Reboot

Reboot the system for the changes to take effect (when you disable SELinux you
must reboot, while logging out and back in is enough to have that user joining
the `kvm` group).

Verify the configuration is correct:

- If installed, disable SELinux:
  ```bash
  $ getenforce
  Disabled
  ```

- The user should be in the `kvm` group:

  ```
  $ groups | grep -ow kvm
  kvm
  ```

### Setup Kata configuration files

These next set of instructions are based on Kata Containers being installed from a package manager.
If `kata-deploy` was used, then the binaries and configuration.toml files will be located at `/opt/kata/` 
and the following instructions will have to be modified.
If the Kata `configuration.toml` file does not exist in `/etc`, create it:

```bash
$ sudo mkdir -p /etc/kata-containers/
$ sudo install -D -m 0640 /usr/share/defaults/kata-containers/configuration.toml /etc/kata-containers/
```

Or, if the file exists, but is not readable by the user:

```bash
$ sudo chown -R a+r /etc/kata-containers/
```

### Disable `vhost-net`

This step will not be required in case you are using Kata based on the latest source.

Disable `vhost-net` in the Kata configuration file, by commenting out the
`disable_vhost_net` line:

```bash
$ sudo sed -i -e 's/^#disable_vhost_net = true/disable_vhost_net = true/' /etc/kata-containers/configuration.toml
```

The above step is required in Kata version 1.10.0-alpha1. In future releases, this should
no longer be required as Kata runtime should handle this automatically for rootless case.
This does mean you will get slightly degraded network performance.

### Modify the Kata image permissions

Upstream qemu needs to have read and write permissions for the kata image in order to 
use it as a nvdimm memory backend, even though in case of Kata, the image is used purely
for a read-only operation. 

Write access on the image should not be required for a read-only access. We have fixed this
 in the qemu packages that we ship with Kata and plan on upstreaming this fix.
(Link to the patch : https://github.com/kata-containers/packaging/blob/master/qemu/patches/4.1.x/0002-memory-backend-file-nvdimm-support-read-only-files-a.patch)

In case you are using a qemu binary provided by your distribution, it is recommended
that you use an initrd instead of rootfs image file. The reason being, you will need to 
add write permissions to the rootfs image for the user running the workload, allowing 
the user to modify the image. 

If you still wish to to use the rootfs image instead of initrd,
 you can change the group ownership of the image by choosing a trusted group and adding
 `rw` permissions for that group. Choose a group where members of the group trust each 
other, as giving write access to the group allows any member of the group to
modify the image.

```bash
$ # Set $GROUP to a group that is trusted.
$ img=$(readlink /usr/share/kata-containers/kata-containers.img)
$ sudo chown -R root:$GROUP /usr/share/kata-containers/$img
$ sudo chmod -R g+rw /usr/share/kata-containers/$img
```
> **Warning:**
>
> You should understand the security implications of the above step before adding write permissions for the image.

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
$ podman run --runtime=/usr/bin/kata-runtime ...
```

or add a `kata` entry in the `[runtimes]` section of the configuration file by
appending the Kata Runtime binary path(s) to the `libpod.conf` file:

```bash
$ echo 'kata = ["/usr/bin/kata-runtime"]' >> ~/.config/containers/libpod.conf
```

and then use `kata` as the runtime name:

```bash
$ podman run --runtime=kata ...
```

> **NOTE:**
>
> A less recommended approach could be to have the absolute `kata-runtime`
> path in the standard `$PATH` location instead of the configuration file. In
> this case it looks up a binary with that name automatically:
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
image name, or add the [configuration
file](https://github.com/containers/libpod/blob/master/install.md#configuration-files).
