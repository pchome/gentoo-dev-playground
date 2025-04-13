# Gentoo playground in qemu for development and compilation
In case you want to build some Gentoo packages on different operating system.<br>
Could theoretically be a good package manager for an project with many dependencies.

## Alternatives
* Gentoo
* Gentoo:Prefix (https://wiki.gentoo.org/wiki/Project:Prefix)

## Source
https://www.gentoo.org/news/2025/02/20/gentoo-qcow2-images.html

## Get qcow2 image
1. visit https://www.gentoo.org/downloads/
2. download  QCOW2 disk (no root pw) [no multilib | systemd] image

## Run script
```sh
#!/bin/sh

# https://distfiles.gentoo.org/releases/amd64/autobuilds/20250322T105044Z/di-amd64-console-20250322T105044Z.qcow2
version=20250322T105044Z
ssh_port=30022

# https://www.qemu.org/docs/master/system/index.html
qemu-system-x86_64 \
        -m 8G -smp 4 -cpu host -accel kvm -vga virtio -smbios type=0,uefi=on \
        -drive if=pflash,unit=0,readonly=on,file=/usr/share/edk2/OvmfX64/OVMF_CODE_4M.qcow2,format=qcow2 \
        -drive file=di-amd64-console-${version}.qcow2 \
        -net nic -net user,hostfwd=tcp::${ssh_port}-:22 \
        -name "gentoo-vm-dev" &
```

## Log in
* `root`
* (no password)

## Network
* guest: `10.0.2.15`
* host:  `10.0.2.2`

### SSH
1. `nano /etc/ssh/sshd_config`
    * `PermitRootLogin yes`
    * `PermitEmptyPasswords yes`
2. `systemctl enable sshd`
3. `systemctl start sshd`

ssh (from host): `ssh -p 20022 root@localhost`

## TL;DR Gentoo basics
See also https://wiki.gentoo.org/wiki/Gentoo_Cheat_Sheet)

### Gentoo binary packages by default
https://wiki.gentoo.org/wiki/Gentoo_Binary_Host_Quickstart

1. `nano /etc/portage/make.conf`
    * `FEATURES="getbinpkg"`
2. `getuto`
    * note: may take a while

NOTE: some binary packages may not meet current USE flags requirements. e.g.:
```
!!! The following binary packages have been ignored due to non matching USE:

    =net-libs/nghttp2-1.65.0-r1 -systemd
    =net-libs/nghttp2-1.65.0-r1 -systemd xml
    =net-libs/nghttp2-1.65.0-r1 xml
```
To fix this just add desired flags to package override.
* `nano /etc/portage/package.use/my-binpkg-flags`
    * `net-libs/nghttp2 xml`

### Installing packages
1. `emerge --sync -q` - sync portage and overlays
2. `emerge -uUD world -vp` - update all packages (optional)
3. `emerge app-portage/eix app-shells/gentoo-bashcomp -q`
4. `eix-update` - update eix database

### Searching packages
* by name: `eix something`
* by description: `eix -S something`
* compact list: `eix -c -I something_installed`

### Using multilib (abi_x86_32 support)
1. `eselect profile list`
    * this will show available profiles, current `no-multilib` profile marked with `*`
2. `eselect profile set 22`
    * this will set multilib profile similar to currently used one
3. `emerge -1 sys-kernel/gentoo-kernel binutils util-linux -q`
    * just in case
4. `nano /etc/portage/package.use/my-multilib`
    * `sys-libs/ncurses abi_x86_32`
    * then `emerge -1 sys-libs/ncurses -q`

### Using overlays (https://wiki.gentoo.org/wiki/Eselect/Repository)
1. `emerge app-eselect/eselect-repository -q`
2. `eselect repository add test git https://github.com/test/test.git`
3. `emaint sync -r test`
4. `eix -c --in-overlay test` - just for example
5. `nano /etc/portage/package.accept_keywords/my-unstable-pkgs`
    * To use unstable or unkeyworded packages:
        * for unstable just add package name: `dev-util/test-pkgname::test` or `=dev-util/test-pkgname-1.2.3::test`
        * for unkeyworded and 9999 add `**` at the end: `dev-util/test-pkgname::test **`
6. `nano /etc/portage/package.use/my-multilib`
    * `dev-util/test-pkgname::test abi_x86_32` - if you also want 32bit support
7. `emerge dev-util/test-pkgname::test -q`

## Get package from installed one
1. `quickpkg dev-util/test-pkgname`
2. use e.g. rsync to download it on host from `/var/cache/binpkgs/dev-util/test-pkgname/*`
    * `rsync -r -e 'ssh -p 20022' root@localhost:/var/cache/binpkgs/dev-util/test-pkgname/* .`
3. repeat with package dependencies if needed
