# Install Kata Containers static binaries

* [Overview](#overview)
* [Installation](#installation)

# Overview

Kata Containers is available as an archive file containing a set of static
binaries. This alternative installation is useful for cloud provisioning
systems which automatically pull the latest static binaries release.

We recommend normal users install using the
[automatic installation method](https://github.com/kata-containers/documentation/tree/master/install)
unless they understand the full implications of installing using the static binaries.

> **WARNING:**
>
> This installation method is **not** recommended for normal users since the
> installation does not use the distribution package manager. This means new
> versions of Kata Containers will **not** automatically install.

# Installation

1. Download the static binaries archive file with the following commands:

   ```bash
   $ arch=$(uname -m)
   $ version=$(curl -Ls -o /dev/null -w %{url_effective} https://github.com/kata-containers/runtime/releases/latest | awk -F\/ '{print $NF}')
   $ file="kata-static-${version}-${arch}.tar.xz"
   $ url="https://github.com/kata-containers/runtime/releases/download/${version}/${file}"
   $ curl -OL "$url"
   ```

1. Unpack the archive:

   ```bash
   $ sudo tar -C / -xvf "$file"
   ```

1. Configure Docker:

   ```bash
   $ sudo /opt/kata/share/scripts/kata-configure-docker.sh "$file"
   ```

1. Install [Docker](docker).
