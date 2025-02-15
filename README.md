# Sailfish OS build environment based on Docker

**Status: Testing**

This is an alternative build environment for Sailfish OS. In contrast
to the official SDKs, it doesn't use
[Scratchbox2](https://github.com/sailfishos/scratchbox2)
(SB2). Instead, it relies on running either native or through QEMU
hardware emulation. When using QEMU emulation, expect very slow
compilation speeds when compared to SB2.

This build environment was created to address official SDK limitations
encountered while packaging Qt 5.15 for Sailfish OS. You may want to
use it if you have a project that is difficult or impossible to
compile using the official SDK.


## How to use it

To use this build environment, you have to have clean sources checked
out. If your sources have compilation artifacts, those could interfere
with the building. As full build process is performed, make sure that
the patches used in `%prep` stage apply.

Until published, Docker builder images have to be generated locally
(see below for instructions). With the builder image ready, go to your
source folder and run (for `sailfishos-i486-4.5.0.19` builder image)
build command similar to:

```
sudo docker run --rm -it -v `pwd`:/source \
   sailfishos-i486-4.5.0.19 buildrpm \
     -r https://repo.sailfishos.org/obs/sailfishos:/chum:/testing/4.5.0.19_i486/
```

In this example, docker container gets access to the sources by its
volume mapping your current directory (`pwd`) to `/source` inside the
container. Container executes `buildrpm` script inside it (see
[sources](scripts/buildrpm)) that handles building RPMs from SPEC in
`/source/rpm`.

There are few options that can be given to `buildrpm` script:

- `-r REMOTE` specify additional repositories for pulling dependencies
  (can be given multiple times);
- `-s SPEC` specify SPEC if there are more than one in `rpm` subfolder
  of the sources. Use just a file basename, as in "test.spec".

If all goes well, RPMs will be created under subfolder `RPMS` of the
sources.

Note that it is recommended to use `--rm` to remove container as soon
as it is finished. On every build, a clean environment is used and all
the dependencies are pulled in again.


## How it works

Inside Docker container, the build is performed in several steps.

First, SPEC file is parsed and all the build requirements are
installed.

Next, RPM build proceeds in a classical way using `rpmbuild`, under a
dedicated user `builder`. For that, folders for building are setup
under `/builder/rpmbuild`. It is possible to debug intermediate steps
through creation of the volume linking to `/builder/rpmbuild` in
docker container.

Before starting the build, your sources are packed into tar.gz (or
bz2, xz, as given in `Source0`) under `/builder/rpmbuild/SOURCES`. In
addition, all files from your sources `rpm` subfolder are copied to
`/builder/rpmbuild/SOURCES`, including all patches.

Then the build proceeds under user `builder` with
`rpmbuild`. `rpmbuild` will unpack the sources, apply patches and
proceed building your packages. After successful build, RPMs are
copied into your source folder under `RPMS` subfolder. For a feedback,
`rpmlint` is applied as well.

Note that, as a new clean environment is created for every build, it
maybe advantageous to debug the build process and reuse the same
container in the case of failure. Easiest is just start the container
with `bash` and execute `buildrpm` already while inside the
container. It is possible to apply the steps done by `buildrpm`
manually as well when inside the container.


## Limitations

The scripts are not handling many cases in a flexible manner provided
by `mb2` from official SDK. For example, during packaging, sources are
packed and then unpacked which wastes time and allocates storage,
until docker container is destroyed. There are probably many other
cases where `mb2` would be able to handle building better.

Due to the way the builder works, Source0 from RPM SPEC is expected to
be in the form `%{name}-%{version}.tar.bz2` (.gz and .zx are supported
as well).


## How to create builder images

Builder Docker images are very easy to create using `makeimage`
script. Make sure you have QEMU setup working for all architectures
that you want to use (see below for QEMU setup, if needed).

Script `makeimage` takes two arguments: SFOS architecture (i486,
aarch64, armv7hl) and SFOS version. Due to issue #1, you would want to
run the script as root:

```
sudo ./makeimage i486 4.5.0.19
```

This will create locally Docker image `sailfishos-i486-4.5.0.19`. This
image can be used for building your packages.


## QEMU setup

To enable QEMU in systemd based Linux, add `qemu-custom.conf` in
`/etc/binfmt.d`. Example:

```
:qemu-aarch64:M::\x7fELF\x02\x01\x01\x00\x00\x00\x00\x00\x00\x00\x00\x00\x02\x00\xb7\x00:\xff\xff\xff\xff\xff\xff\xff\x00\xff\xff\xff\xff\xff\xff\xff\xff\xfe\xff\xff\xff:/usr/bin/qemu-aarch64:F
:qemu-arm:M::\x7fELF\x01\x01\x01\x00\x00\x00\x00\x00\x00\x00\x00\x00\x02\x00\x28\x00:\xff\xff\xff\xff\xff\xff\xff\x00\xff\xff\xff\xff\xff\xff\xff\xff\xfe\xff\xff\xff:/usr/bin/qemu-arm:F
```

After that, restart systemd service:
```
systemctl restart systemd-binfmt
```

You should see formats registered in `/proc`:
```
# cat /proc/sys/fs/binfmt_misc/qemu-aarch64
enabled
interpreter /usr/bin/qemu-aarch64
flags: F
offset 0
magic 7f454c460201010000000000000000000200b700
mask ffffffffffffff00fffffffffffffffffeffffff
```

See References below for how to test QEMU support using Docker images.


## References

- SFOS Docker scripts: https://github.com/CODeRUS/docker-sailfishos-baseimage
- Docker architectures: https://github.com/docker-library/official-images#architectures-other-than-amd64
- QEMU setup: https://wiki.gentoo.org/wiki/Embedded_Handbook/General/Compiling_with_qemu_user_chroot
- QEMU tests with Docker: https://github.com/multiarch/qemu-user-static