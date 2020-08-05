# Kata Containers with SGX

- [Install Host kernel with SGX support](#install-host-kernel-with-sgx-support)
- [Install Guest kernel with SGX support](#install-guest-kernel-with-sgx-support)
- [Run Kata Containers with SGX enabled](#run-kata-containers-with-sgx-enabled)

Intel® Software Guard Extensions (SGX) is a set of instructions that increases the security
of applications code and data, giving them more protections from disclosure or modification.
At the time of writing this document, SGX patches have not landed on the Linux kernel
project and `cloud-hypervisor` 0.8.0 doesn't support SGX, so specific versions of `cloud-hypervisor`,
guest and host kernel must be installed to enable SGX.

## Install Host kernel with SGX support

The following commands were tested on fedora 32, they might work on other distros too.

```sh
git clone --depth=1 https://github.com/intel/kvm-sgx
pushd kvm-sgx
cp /boot/config-$(uname -r) .config
yes n | make oldconfig
# In the following step, enable: INTEL_SGX and INTEL_SGX_VIRTUALIZATION
make menuconfig
make -j$(($(nproc)-1)) bzImage
make -j$(($(nproc)-1)) modules
sudo make modules_install
sudo make install
popd
sudo reboot
```

_Note: Run: `mokutil --sb-state` to check whether secure boot is enabled, if so, you will need to sign the kernel._

Once you have restarted your system with the new brand Linux Kernel with SGX support, run
the following command to make sure it's enabled. If the output is empty, go to the BIOS
setup and enable SGX manually.

```sh
grep -o sgx /proc/cpuinfo
```

## Install Guest kernel with SGX support

Install the guest kernel in the kata-containers directory, this way it can be used to run
Kata containers.

```sh
curl -LOk https://github.com/devimc/kvm-sgx/releases/download/v0.0.1/kata-virtiofs-sgx.tar.gz
sudo tar -xf kata-virtiofs-sgx.tar.gz -C /usr/share/kata-containers/
sudo sed 's|kernel =|kernel = "/usr/share/kata-containers/vmlinux-virtiofs-sgx.container"|g' \
/usr/share/defaults/kata-containers/configuration.toml
```

## Run Kata Containers with SGX enabled

At the time of writing this document, `cloud-hypervisor` 0.9.0 has not been released, so
`cloud-hypervisor` at commit `840445096af8ae736b0163fe73aa22f1908e2930` should be installed,
for more information about how to build `cloud-hypervisor` go to its [repository][1].
Once `cloud-hypervisor` is installed, run a Kata container using ctr and check that the SGX
devices have been correctly created under `/dev/sgx`:

```sh
ctr run --mount type=bind,src=/dev/sgx/,dst=/dev/sgx/,options=rbind:ro --runtime io.containerd.run.kata.v2 -t --rm docker.io/library/busybox:latest hello sh
# ls /dev/sgx
enclave provision virt_epc
```

[1]: github.com/cloud-hypervisor/cloud-hypervisor/
