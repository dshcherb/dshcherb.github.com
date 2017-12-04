---
layout: post
title: "Running QEMU/KVM Virtual Machines in Unprivileged LXD Containers"
date: 2017-12-04
description: Notes on how to create a fast and clean setup
---

# Introduction

Sometimes you only have your laptop or a very limited environment but need to test quite complex setups that involve multiple hosts and network segments. For some purposes containers are good enough and LXD allows creating containers that give similar abstractions to full virtual machines. There are other scenarios where virtual machines are needed.

# Containers and Kernel Isolation

In a simplified view, containers can be described as processes isolated via Linux kernel mechanisms. Mechanisms mentioned below can be used to create different varieties of isolation for containers in general - there is no strict definition on what should or should not be enabled:

* 7 kernel namespaces (mount, pid, user, ipc, network, uts, cgroup);
* cgroups - process grouping, resource limits (cpu, ram, network, block IO), device special file access (mknod, open), freezing tasks;
* [capabilities](http://man7.org/linux/man-pages/man7/capabilities.7.html) (e.g. CAP_SYS_ADMIN is required to do [mount(2)](http://man7.org/linux/man-pages/man2/mount.2.html) system call);
* LSM (AppArmor);
* [seccomp](http://man7.org/linux/man-pages/man2/seccomp.2.html) - system call filtering using BPF by system call number and arguments;
* file system isolation: a container usually gets its own root file system. Root file system is a [process property](http://elixir.free-electrons.com/linux/v4.14.3/source/include/linux/sched.h#L772) which can be changed using [pivot_root(2)](http://man7.org/linux/man-pages/man2/pivot_root.2.html) and is inherited by child processes;
* Process limits set via [prlimit(2)](http://man7.org/linux/man-pages/man2/prlimit.2.html).

# QEMU/KVM

QEMU is a **userspace program** to run virtual machines - an emulator. [KVM is a kernel module](http://elixir.free-electrons.com/linux/v4.14.3/source/virt/kvm/kvm_main.c) that allows userspace processes to utilize Intel (VT-x) or AMD (AMD-V) virtualization technologies present in almost every modern CPU to avoid some overhead associated with full emulation done by QEMU (memory management, interrupts, timers etc.). [QEMU can utilize KVM](https://git.qemu.org/?p=qemu.git;a=blob;f=accel/kvm/kvm-all.c;h=f290f487a573adc8632165a3d8cef3a80e77c5c5;hb=HEAD) using [ioctl(2)](http://man7.org/linux/man-pages/man2/ioctl.2.html) [interface it provides](http://elixir.free-electrons.com/linux/v4.14.3/source/Documentation/virtual/kvm/api.txt) via a character special file /dev/kvm. This is what people call "QEMU/KVM" or just "KVM".

There are other technologies that remove even more emulation overhead:

* vhost is used to move user-space device emulation implementation of the data plane to the kernel space (vhost, vhost_net, vhost_scsi, vhost_vsock [kernel modules](http://elixir.free-electrons.com/linux/v4.14.3/source/drivers/vhost)) and avoid system call overhead. vhost_vsock specifically allows a guest to use special sockets to communicate with a hypervisor (host) more efficiently and with less modifications than with serial devices;
* vhost-user is used to move user-space device emulation implementation of the data plane to a different userspace application. This is mainly used for user-space driver implementations that bypass the kernel stack completely - [DPDK-](https://software.intel.com/en-us/articles/data-plane-development-kit-vhost-user-client-mode-with-open-vswitch) or [SPDK-based](http://www.spdk.io/doc/vhost.html) applications are a good example. For networking this is used in [OVS](http://docs.openvswitch.org/en/latest/intro/install/dpdk/) or [Snabb](http://www.virtualopensystems.com/en/solutions/guides/snabbswitch-qemu/) to speed up packet processing by avoiding extra context and mode-switches for interrupt processing. For storage the problem is similar: with fast NVMe devices too many interrupts are generated in which case CPU becomes a bottleneck. Such technologies utilize memory locking and huge pages with dedicated threads isolated from [load-balancing by the kernel scheduler](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=47b8ea7186aae7f474ec4c98f43eaa8da719cd83) to process ring buffers.

# Unprivileged Containers

With the above in mind it might seem like we absolutely need a privileged container to run virtual machines. This is not entirely true because there are different kinds of privileges. In the Linux world there are privileged processes as [capabilities(7)](http://man7.org/linux/man-pages/man7/capabilities.7.html) mentions:

> For the purpose of performing permission checks, traditional UNIX implementations distinguish two categories of processes: privileged processes (whose effective user ID is 0, referred to as superuser or root), and unprivileged processes (whose effective UID is nonzero). Privileged processes bypass all kernel permission checks, while unprivileged processes are subject to full permission checking based on the process' credentials (usually: effective UID, effective GID, and supplementary group list).

[LXC security page](https://linuxcontainers.org/lxc/security/#privileged-containers) is fairly clear and coherent with that definition but for containers:

> Privileged containers are defined as any container where the container uid 0 is mapped to the host's uid 0. In such containers, protection of the host and prevention of escape is entirely done through Mandatory Access Control (apparmor, selinux), seccomp filters, dropping of capabilities and namespaces.

> Unprivileged containers are safe by design. The container uid 0 is mapped to an unprivileged user outside of the container and only has extra rights on resources that it owns itself.

It is clear that in an unprivileged container a separate **user namespace** is created while in a privileged container this is not the case (in other words, CLONE_NEWUSER is either used or not used in either clone(2) or unshare(2) system calls).

How does that help with creating virtual machines?

# Virtual Machines in Unprivileged Containers

* clone(2) or fork(2) system calls can be used to create new processes and exec family of system calls can be used to execute new binaries in unprivileged containers just fine and a QEMU binary falls into that category;
* QEMU/KVM expects a few kernel modules to be loaded, mainly `kvm`, `kvm_intel` or `kvm_amd` and `tap`. Performance-wise, depending on your setup `vhost`, `vhost_net`, `vhost_scsi` and `vhost_vsock` modules. If VFIO needs to be used `vfio` and `vfio-pci` at least are also needed but this requires host sysfs access as well which is a bit more involved (you may also need to load and control a hardware device driver via its own character special files);
* QEMU/KVM needs access to a number of character special files `/dev/kvm`, `/dev/net/tun`, `/dev/vhost-net`, `/dev/vhost-scsi`, `/dev/vhost-vsock`. For VFIO `/dev/vfio/vfio` and potentially other driver-specific character special files.
* Libvirt daemon manages QEMU processes and they go through a daemonization procedure to stay running even if libvirtd exits. Libvirt uses some kernel functionality, including `bridge` module and cgroups;

Other than VFIO-related modules and character files or customizations required for usage of functionality such as huge pages, there is not a lot to enable.

Character special files can be used to run module-specific operations via a generic [ioctl(2)](http://man7.org/linux/man-pages/man2/ioctl.2.html) interface. Some ioctls require certain capabilities(7) to be present but this is module-specific - there has to be [special code](http://elixir.free-electrons.com/linux/v4.14.3/source/drivers/tty/tty_io.c#L2564) in a kernel module to enforce that. There is no explicit access control besides file permissions or, if present, LSM-based mandatory access control (e.g. AppArmor).

For that reason, provided that modules are loaded by a privileged user before a container starts or at its runtime, there are no barriers to running accelerated virtual machines in containers.

# Outdated Requirements

There are some [outdated requirements](https://libvirt.org/git/?p=libvirt.git;a=blob;f=src/qemu/qemu.conf;h=6ec893ac1f61d4011a31a90be02b827436edebca;hb=6380fb9795cf921b96e1ad235a0cab78e318299d#l450) with regards to character special files which should be ignored:

* `/dev/kqemu`
* `/dev/rtc`
* `/dev/hpet`

[kqemu is long gone](https://wiki.qemu.org/Documentation/KQemu), same with [rtc and hpet](https://git.qemu.org/?p=qemu.git;a=commitdiff;h=25f3151ece1d5881826232bebccc21b588d4e03e;hp=d800040fb47fe4500d1f8bf604b9fd129bda9419).

# Breaking Ground

LXD supports a number of useful ways to configure containers, including a way to [preseed cloud-init](http://lxd.readthedocs.io/en/latest/cloud-init/) network-data or user-data for Ubuntu cloud-images that have certain instrumentation in container templates. The following functionality will be used:

* LXD profiles;
* cloud-image templates and cloud-init network-data and user-data;
* LXD-side pre-loading of kernel modules before containers are started;
* ability to pre-create character special files for containers;
* storage pools.

Below is the template that can be used to configure an LXD profile:

{% highlight yaml %}
{% raw %}

config:
  security.nesting: "true"
  linux.kernel_modules: iptable_nat, ip6table_nat, ebtables, openvswitch, kvm, kvm_intel, tap, vhost, vhost_net, vhost_scsi, vhost_vsock
  user.network-config: |
    version: 1
    config:
      - type: physical
        name: eth0
        subnets:
          - control: manual
      - type: bridge
        name: br0
        bridge_interfaces:
          - eth0
        params:
          bridge_stp: 'off'
          bridge_fd: 1
        subnets:
           - type: dhcp
  user.network-config: |
    version: 1
    config:
      - type: physical
        name: eth0
        subnets:
           - type: manual
             control: auto
      - type: bridge
        name: br0
        subnets:
           - type: dhcp
             control: auto
        bridge_interfaces:
          - eth0
        params:
          bridge_stp: 'off'
          bridge_fd: 1

  user.user-data: |
    #cloud-config
    runcmd:
    - dhclient eth0
    - apt update
    - apt install -yqq bridge-utils libvirt-bin qemu-kvm
    - ip addr flush eth0
    - ifup -a
    - mv /etc/network/interfaces.d/50-cloud-init.cfg /etc/network/interfaces
devices:
  kvm:
    path: /dev/kvm
    type: unix-char
  root:
    path: /
    pool: default
    type: disk
  tun:
    path: /dev/net/tun
    type: unix-char
  vhost-net:
    path: /dev/vhost-net
    type: unix-char
    mode: 0600
  vhost-scsi:
    path: /dev/vhost-scsi
    type: unix-char
    mode: 0600
  vhost-vsock:
    path: /dev/vhost-vsock
    type: unix-char

{% endraw %}
{% endhighlight yaml %}


{% highlight bash %}
{% raw %}

# LXD migrates to snaps so on a clean machine remove that and install a snap
# sudo apt purge lxd && sudo snap install lxd
# alternatively:
# sudo snap install lxd && sudo lxd.migrate

lxc profile create vmct

# paste the profile code above after this command
cat | lxc profile edit vmct

# profiles will get merged - vmct will take precedence
lxc init ubuntu:xenial <container-name> -p default -p vmct

# optionally create a storage pool, in this case this is a directory pool
# but if you use btrfs, ZFS or LVM to create volumes you can use those instead
lxc storage create <poolname> dir source=<hostdir>
lxc storage volume create <poolname> <volname>
lxc storage volume attach <poolname> <volname> <container-name> <mnt-path-in-container>

# after you start it wait for some time as packages will need to be
# installed. bridge-utils package is not installed by default - hence
# the magic with runcmd in user-data.
# runcmd will only be executed on first boot of a given container
# ifup will error out on bridging the bridge interface up if
# bridge-utils package is not installed

lxc start <container-name>

{% endraw %}
{% endhighlight bash %}

Basic `virt-host-validate` checks pass when used with the profile above.

{% highlight bash %}
{% raw %}

root@<container-name>:~# virt-host-validate
  QEMU: Checking for hardware virtualization                 : PASS
  QEMU: Checking if device /dev/kvm exists                   : PASS
  QEMU: Checking if device /dev/kvm is accessible            : PASS
  QEMU: Checking if device /dev/vhost-net exists             : PASS
  QEMU: Checking if device /dev/net/tun exists               : PASS
  QEMU: Checking for cgroup 'memory' controller support      : PASS
  QEMU: Checking for cgroup 'memory' controller mount-point  : PASS
  QEMU: Checking for cgroup 'cpu' controller support         : PASS
  QEMU: Checking for cgroup 'cpu' controller mount-point     : PASS
  QEMU: Checking for cgroup 'cpuacct' controller support     : PASS
  QEMU: Checking for cgroup 'cpuacct' controller mount-point : PASS
  QEMU: Checking for cgroup 'devices' controller support     : PASS
  QEMU: Checking for cgroup 'devices' controller mount-point : PASS
  QEMU: Checking for cgroup 'net_cls' controller support     : PASS
  QEMU: Checking for cgroup 'net_cls' controller mount-point : PASS
  QEMU: Checking for cgroup 'blkio' controller support       : PASS
  QEMU: Checking for cgroup 'blkio' controller mount-point   : PASS
  QEMU: Checking for device assignment IOMMU support         : PASS
  QEMU: Checking if IOMMU is enabled by kernel               : WARN (IOMMU appears to be disabled in kernel. Add intel_iommu=on to kernel cmdline arguments)
   LXC: Checking for Linux >= 2.6.26                         : PASS
   LXC: Checking for namespace ipc                           : PASS
   LXC: Checking for namespace mnt                           : PASS
   LXC: Checking for namespace pid                           : PASS
   LXC: Checking for namespace uts                           : PASS
   LXC: Checking for namespace net                           : PASS
   LXC: Checking for namespace user                          : PASS
   LXC: Checking for cgroup 'memory' controller support      : PASS
   LXC: Checking for cgroup 'memory' controller mount-point  : PASS
   LXC: Checking for cgroup 'cpu' controller support         : PASS
   LXC: Checking for cgroup 'cpu' controller mount-point     : PASS
   LXC: Checking for cgroup 'cpuacct' controller support     : PASS
   LXC: Checking for cgroup 'cpuacct' controller mount-point : PASS
   LXC: Checking for cgroup 'devices' controller support     : PASS
   LXC: Checking for cgroup 'devices' controller mount-point : PASS
   LXC: Checking for cgroup 'net_cls' controller support     : PASS
   LXC: Checking for cgroup 'net_cls' controller mount-point : PASS
   LXC: Checking for cgroup 'freezer' controller support     : PASS
   LXC: Checking for cgroup 'freezer' controller mount-point : PASS

{% endraw %}
{% endhighlight bash %}
