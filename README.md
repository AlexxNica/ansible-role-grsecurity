# ansible-role-grsecurity

Build and install Linux kernels with the grsecurity patches applied.
Supports "test" and "stable" grsecurity patches. Using the "stable"
patches will [require subscription](https://grsecurity.net/business_support.php).

These configurations were developed by [Freedom of the Press Foundation] for
use in all [SecureDrop] instances. Experienced sysadmins can leverage these
configuration roles to compile custom kernels for SecureDrop or non-SecureDrop
architecture.

## Requirements

Only Debian and Ubuntu are supported, but that's mostly due to lack of testing,
rather than an inherent deficiency in the configs. If you plan to the stable patches,
you'll need to sign up for a [grsecurity subscription](https://grsecurity.net/business_support.php).
For compiling the kernel, 2GB and 2 VCPUs is plenty. Depending on the config options
you specify, the compilation should take two to three hours on that hardware.

## Role structure

There are three roles contained in this repository:

* build-grsec-metapackage
* build-grsec-kernel
* install-grsec-kernel

The metapackage role is SecureDrop-specific, so you can ignore it if you're compiling
for non-SecureDrop machines. The build role will download the Linux kernel source tarball
and the latest grsecurity patch and prepare the system for a manual build. The install role
expects a .deb package filepath on the Ansible controller, and will install that package
on the target host.

## Role variables

### build-grsec-kernel
```yaml
# Can be "stable" or "test". Note that stable patches
# requires authentication to download. See the grsecurity
# blog for more information: https://grsecurity.net/announce.php
grsecurity_build_patch_type: test

# The default "manual" strategy will prep a machine for compilation,
# but stop short of configuring and compiling. You can instead choose
# to compile a kernel based on a static config shipped with this role,
# for a "Look ma, no hands!" kernel compilation. See the "files" dir
# for possible config options. The var below is interpolated as
# "config-{{ grsecurity_build_strategy }}" when searching for files.
grsecurity_build_strategy: manual

# When building for installation on Ubuntu, one should include the
# overlay to ensure that Ubuntu-specific options for AppArmor work.
# Honestly this needs a lot more testing, so leaving off by default.
grsecurity_build_include_ubuntu_overlay: false

# Parent directory for storing source tarballs and signature files.
grsecurity_build_download_directory: "{{ ansible_env.HOME }}/linux"

# Extracted source directory, where you should run `make menuconfig`.
grsecurity_build_linux_source_directory: >-
  {{ grsecurity_build_download_directory }}/linux-{{ linux_kernel_version }}

grsecurity_build_gpg_keyserver: hkps.pool.sks-keyservers.net

# Assumes 64-bit (not reading machine architecture dynamically.)
grsecurity_build_deb_package: >-
  linux-image-{{ linux_kernel_version }}-grsec_10.00.{{ grsecurity_build_strategy }}_amd64.deb

# Using ccache can dramatically speed up subsequent builds of the
# same kernel source. Disable if you plan to build only once.
grsecurity_build_use_ccache: true
```

### install-grsec-kernel
```yaml
# This var is required, but can't be known ahead of time. Build the Debian package
# with the grsecurity build role, then set var to the filepath to the .deb.
# The role will fail if this var is not updated.
grsecurity_install_deb_package: ''

# For easier console recovery and debugging, the GRUB timeout value (default: 5)
# can be overridden here. Without a lengthier timeout, it can be very difficult
# to get into the GRUB menu and select a working kernel to boot.
grsecurity_install_grub_timeout: 5

# paxctld is a better alternative than paxctl for maintaining the PaX flags on binaries.
# The paxctld role isn't a dependency yet, so assume the paxctl approach is safest.
# If you're using the paxctld role, set this to false.
grsecurity_install_set_paxctl_flags: true

# Location where the .deb files will be copied on the target host, prior to install.
grsecurity_install_download_dir: /usr/local/src

# The role will skip installation if the kernel version, e.g. "4.4.2-grsec",
# of the deb package matches that of the target host, provided the checksum
# for the deb file is the same. If you want to reinstall the same kernel version,
# for example while developing a new kernel config, set this to true.
grsecurity_install_force_install: false
```

The primary components of interest in this repository are:

1. `securedrop-grsec`, the kernel metapackage
2. `build.md`, the guide for building the grsecurity kernel for SecureDrop

## Quickstart
Use the Vagrant VMs to build a grsecurity-patched kernel:

```
vagrant up grsec-build
vagrant ssh
cd linux/linux-<version>
make menuconfig
export CONCURRENCY_LEVEL="$(nproc)"
export PATH="/usr/lib/ccache:$PATH" # recommended if you plan to recompile
fakeroot make-kpkg --initrd kernel_image
```

[Freedom of the Press Foundation]: https://freedom.press
[SecureDrop]: https://securedrop.org
[grsecurity]: https://grsecurity.net/
