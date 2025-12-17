# Linux Build SDK

This document describes two blocks for creating a Linux Build SDK:

1. build infrastructure as a set of meta-data, and configuration used for creating a full rootfs
   if all compilers, libraries and tools for building Erlang/OTP binaries from source code.
2. runtime infrastructure which describes where and how the rootfs will be executed

## build infrastructure

The following descriptions defines what is called _build infrastructure_ which is a set of meta data used
for creating all tools needed in order to build Erlang/OTP binaries for Linux platform.

We plan to use Yocto Project in order to make specific rootfs for each build supported build architecture and
having all tools needed to perform native builds.

That means for each build architecture Erlang/OTP source code will be natively build using the build
[runtime infrastructure](#runtime-infrastructure). In this stage there is no cross compilation planned.
That is for minimizing host contamination issues and not because cross compilation is not technically supported and feasible.

As a framework Yocto Project gives us some important non-functional requirements:

 - support all common microprocessor architectures and it is easy to support additional architectures
 - the capability to patch any component with security fixes, additional features
 - quickly upgrade component versions
 - SBOM and license reports for all component used
 - well supported by community and also from companies and consultancies
 - provide a complete set of debug tools to investigate any build issues that may arise

And the following functional requirements:

 - possibility to install multi versions of components. E.g.: install different openssl versions without conflicting with OS
 - produce an Erlang/OTP binary for Linux that runs on every Linux box
 - have generic support configuration for following architectures (more will be available in the future):
   - x86-64-v3
   - x86-64-v4
   - aarch64
 - possibility to tune build flags for specific architecture

One of the key points is to select the right Yocto Project version to use. As we are going to build rootfs
for building Erlang/OTP, it is important to select stable and well maintained components. Given that, the Yocto Project
release called `scarthgap` (5.0) is a good starting point. It is based on the following components (a full list is available
here [Release notes for 5.0 (scarthgap)](https://docs.yoctoproject.org/migration-guides/release-notes-5.0.html)):

 - Linux kernel 6.6
 - gcc 13.2
 - glibc 2.39
 - musl 1.2.4
 - LLVM 18.1
 - openssl 3.2.4
 - ncurse 6.4

scarthgap was released in April 2024 as LTS support level until April 2028.

After 2028 we will have to evaluate a new Yocto LTS release or extend the scarthgap support by ourself. This is
a future decision point because we can not get the latest Yocto version but something stable, mainly related to libc.
As one of our aim is to produce an Erlang/OTP binary release and we cannot build it with latest libc where that
version could not be widely used yet by most of mainstream Linux distributions. So, the main takeaway is to be conservative.

In practical terms a new yocto layer called `meta-erlang-build` will be defined. This layer will have recipes for building four
kind of artifacts for each supported architecture:

 - A rootfs tarball, it is a full working image without linux kernel and with all userland tools
   for building Erlang/OTP binaries

 - An OCI container, it is the same rootfs described as above but packaged as OCI container. This is
   the main output required for the rest of build process

 - A SBOM and license report about all components present in the final rootfs

 - A SDK tarball for cross-compilation, this is optional and not required

As Yocto has the ability to build for many architectures we will reuse the same infrastructure and changing
only the MACHINE type and we will be able to build for all planned architectures. This change is configuration based.

Each rootfs will have all required libraries for build Erlang/OTP, the following libraries will be provided:

 - openssl
 - ncurses
 - wxwidgets
 - lksctp (?)
 - unixodbc (?)

Also the following libc flavours will be available (one rootfs for each):

 - libc
 - musl

The rootfs will provide all libraries for static linkage. That way an Erlang/OTP build can enable static options to use the correct
libraries.

With exception of libc and musl, that are linked directly with a Yocto release and defines the flavor of the entire rootfs,
it is a requirement to have Erlang/OTP dependency libraries installed in an alternative path (e.g.: _/usr/local_). This way opens the possibility
to configure Erlang/OTP pointing it to the exactly library version for build.

The layer meta-erlang-build also implements a image recipe type for building the rootfs. This recipe is where we will declare
all components that we want as a final rootfs.

The layer is the heart of this build infrastructure because it will define all the base tools used when building Erlang/OTP and we have
to have a well understand about all the components, configuration in order to fully control the process.

## runtime infrastructure

Given that the previous section takes care about producing a working rootfs for each target architecture, this section
describes how we use the rootfs.

As each rootfs is architecture specific, we need some sort of emulation to execute each rootfs. We also want to instantiate 
this environment from a CI point of view. Actually as each rootfs is provided also as OCI container, one possibility
is to use the [binfmt](https://github.com/tonistiigi/binfmt) to configure the right binfmt_misc from Linux kernel and call
the respective qemu-user mode for running each program inside the rootfs container.

The components involved in this idea is the following:

 - [binfmt_misc](https://docs.kernel.org/admin-guide/binfmt-misc.html), is a Linux kernel feature that calls interpreter
   matching each binary-type which is some bytes at the beginning of each file. References:  and https://man7.org/linux/man-pages/man5/binfmt.d.5.html

 - [QEMU](https://www.qemu.org/) (actually the qemu-user-mode variation) as the interpreter to be called by Linux kernel
   when it detects a configured binary-type matching

 - [binfmt](https://github.com/tonistiigi/binfmt) as the mechanism to configure docker containers with QEMU user mode. binfmt plays configuring
   many binary types and theirs respective handlers.

 - docker, instantiating (docker run) the respective rootfs docker container.


 Having all above components configured we will be able to use any CI to run an Erlang/OTP build script inside rootfs container and after the build
 the output will be a valid binary ready to be installed in the target Linux system.