# cockroachdb/bincheck

[![Build Status](https://travis-ci.org/benesch/bincheck.svg?branch=master)](https://travis-ci.org/benesch/bincheck)
[![Build status](https://ci.appveyor.com/api/projects/status/6u82wqnbjdrsvggu?svg=true)](https://ci.appveyor.com/project/benesch/bincheck)

bincheck verifies the sanity of CockroachDB release binaries. At present, the
sanity checks are:

  * starting a one-node server and running a simple SQL query, and
  * verifying the output of `cockroach version`.

## Testing a new release

Update the version in [VERSION] appropriately, commit, and push! macOS and Linux
binaries are verified by TravisCI; Windows binaries are verified by AppVeyor.

## The nitty-gritty

### Overview

For the macOS and Windows binaries, the mechanics involved are simple. We ask
Travis or AppVeyor, respectively, to spin us up a macOS or Windows VM, download
the appropriate pre-built `cockroach` binary, and run our sanity checks.

Linux is more complicated, not out of necessity, but out of ambition. We co-opt
the Linux verification step to additionally test support for pre-SSE4.2
CPUs<sup>†</sup>. This requires emulating such a CPU, and Linux is the only
operating system that is feasible to run under emulation. Windows and macOS have
prohibitively slow boot times, and, perhaps more importantly, Windows and macOS
install media are not freely available.

So, with the help of [Buildroot], an embedded Linux build manager, this
repository ships an [8.7MB Linux distribution][linux-image] that's capable of
running under [QEMU] and launching our pre-built CockroachDB binaries. To verify
the Linux binaries, we first boot this Linux distribution on an emulated
pre-SSE4.2 CPU (`qemu-system-x86_64 -cpu qemu64,-sse4.2`), then run our sanity
checks from inside this VM.

<sup>†</sup><small>SSE4.2 support is particularly important in CockroachDB,
since RocksDB internally uses a [CRC32C checksum][crc32c] to protect against
data corruption. SSE4.2 includes hardware support for computing CRC32C
checksums, but is only available on CPUs released after November 2008. Producing
a binary that can dynamically switch between the SSE4.2 and non-SSE4.2
implementations at runtime [has proven difficult][issue-15589].
</small>

### Updating the Linux image

After [installing Buildroot][buildroot-install]:

```shell
$ make qemu_x86_64_glibc_defconfig BR2_EXTERNAL=${BINCHECK-REPO}/buildroot
$ make menuconfig  # Only if configuration changes are necessary.
$ make
$ cp output/images/bzImage ${BINCHECK-REPO}/images/qemu_x86_64_glibc_bzImage
```

At the time of writing, `qemu_x86_64_glibc_defconfig` instructed Buildroot to
build a Linux 4.9 kernel, install Bash and OpenSSH into userland, and configure
sshd to boot at startup and allow passwordless `root` authentication.
`/etc/fstab` is modified to mount the first hard drive at `/bincheck`, assuming
it's a FAT volume.

[buildroot-install]: https://buildroot.org/download.html
[issue-15589]: https://github.com/cockroachdb/cockroach/issues/15589
[linux-image]: ./images/qemu_x86_64_glibc_bzImage
[Buildroot]: https://buildroot.org
[CRC32C]: http://www.evanjones.ca/crc32c.html
[QEMU]: http://qemu.org
[VERSION]: ./VERSION