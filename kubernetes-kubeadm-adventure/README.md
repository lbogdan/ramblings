*published on TODO: add date*

> **Tip**
>
> To see the table of contents, you can click on the outline button <img alt="outline button" src="images/outline.png" width="24"> in the top-right corner.

# Adventures in Kubernetes Land: Create and Manage Bare-Metal, Production-Ready Clusters Using `kubeadm`

## Introduction, Motivation, Approach

First of all, let me quickly introduce myself: I am an infrastructure-first, development-second kind of person, working with Kubernetes since around 1.9 (released Oct 2017, so almost five years ago), on a daily basis. During this time, I've managed both provider-managed clusters (EKS, GKE), and bare-metal ones. As this was so long ago, back then `kubeadm` was the only "certified" tool to create a bare-metal Kubernetes cluster.

So why am I writing this? Because creating a bare-metal, production-ready Kubernetes cluster using `kubeadm` is not a trivial task - you have to make some upfront decisions that are not obvious if you don't have enough experience operating bare-metal clusters - in other words, you don't know what you don't know. The official documentation only goes so far, usually explaining **how** to do things, but not **why**. Most articles online do the same thing, skimming over important details, and ending a long way of a production-ready cluster, usually not even talking about Day 2 operations. I've yet to find a comprehensive article on the topic; the series that comes closest, which finally pushed me to write this, [Bare-metal Kubernetes, Part I: Talos on Hetzner](https://datavirke.dk/posts/bare-metal-kubernetes-part-1-talos-on-hetzner/), uses [Talos](https://www.talos.dev/), which is a dedicated Kubernetes Linux distribution - and a very big decision to make in adopting it -, so we'll just keep to a generic Linux distribution and `kubeadm` for now.

We'll have a hands-on approach (of course!), starting with the most simplest, one-node bare-metal cluster, explore its drawbacks, and slowly build on that - both in configuration options and number of nodes. During our journey, be prepared to make mistakes, troubleshoot and fix them! The final goal is to reach a production-ready, HA control plane cluster, being able to publicly expose applications on `https` endpoints, with distributed storage, basic monitoring and logging, and upgrading it between minor Kubernetes versions.

While I will assume you are familiar with managing Linux (we'll use Ubuntu) systems, I will try my best to explain and give external references to the concepts we'll be using. That being said, let's dive in!

## Prerequisites: Hardware and Software

Before anything, let's first take a look at [`kubeadm` prerequisites](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/#before-you-begin). From my experience, the minimum hardware requirement of 2 cores (/ vCPUs) and 2GB of RAM is not enough to run anything meaningful. My recommendations are:

- for a one-node cluster, at least 4 cores and 8GB of RAM, as you'll be running both the Kubernetes control-plane components and your own workloads on the same server;

- for a multi-node cluster, at least 4 cores and 4GB or RAM.

Fast storage (NVMe) is also recommended on the control-plane nodes, as `etcd`, Kubernetes' data store, is [very sensitive to disk write latency](https://etcd.io/docs/v3.3/op-guide/hardware/#disks).

Of course, all these highly depend on the size of the cluster, both in the number of nodes, and the number, size, and dynamicity of your workloads. To give you an idea, the clusters I've been managing over time have 64GB, >=8 cores control-plane nodes, and >=64GB, >=16 cores worker nodes, all with NVMe M.2 PCIE drives.

For this series I'll be using some [Hetzner VPSes](https://www.hetzner.com/cloud) with assorted hardware specifications. Note that managing the hardware is outside the scope of this series.

The only software prerequisite is a Linux distribution that you're familiar with, using `systemd`, a fairly recent kernel version, [cgroup v2](https://docs.kernel.org/admin-guide/cgroup-v2.html), and having packages for Kubernetes tools and components (`kubeadm`, `kubectl`, `kubelet`) and container runtime (either `containerd` or `cri-o`). I've been using Ubuntu since 12.04 or so (released more than 10 years ago), so it is my distribution of choice. For this I'll be using the current LTS release, Ubuntu 22.04.3.

For now, we're assuming the server has only one network interface, with a publicly accessible IPv4 IP address, and no firewall.

I'll go ahead and create a Hetzner VPS (TODO: add specs) (for now, we'll add more nodes eventually) with a minimal server install of Ubuntu 22.04.3 LTS. I'll name it `test-cp-0`, which will also be set as the hostname. This is important because, by default, the Kubernetes node name will be set to the server's hostname.

```sh
# from a ssh session to root@test-cp-0
lsb_release -a
# output:
# No LSB modules are available.
# Distributor ID: Ubuntu
# Description:    Ubuntu 22.04.3 LTS
# Release:        22.04
# Codename:       jammy
hostname -f
# output:
# test-cp-0
```

> **Note**
> Although not best practice, I'll be running all commands as the `root` user, for convenience sake.

## Initial Server Setup

First of all, we'll do some setting-up not directly related to Kubernetes:

- Upgrade packages:

```sh
apt-get -q update
apt-get -qy upgrade
# if shown, press ESC at the restart services dialog
```

- Remove unneeded packages:

```sh
apt-get -qy purge multipath-tools polkitd udisks2
# this will leave packages no longer required, which can be `autoremove`d
apt-get -qy autoremove
```

- Install a newer kernel. Ubuntu 22.04 comes with `5.15.0` by default, which is quite old at this point. We'll use the Ubuntu kernel [mainline](https://wiki.ubuntu.com/Kernel/MainlineBuilds) [builds](https://kernel.ubuntu.com/~kernel-ppa/mainline/) and install the current stable version, `6.5.3`. What could go wrong, right?!

  ```sh
  uname -a && pwd
  # output:
  # Linux test-cp-0 5.15.0-79-generic #86-Ubuntu SMP Mon Jul 10 16:07:21 UTC 2023 x86_64 x86_64 x86_64 GNU/Linux
  # /root
  mkdir kernel && cd kernel
  wget https://kernel.ubuntu.com/~kernel-ppa/mainline/v6.5.3/amd64/linux-headers-6.5.3-060503-generic_6.5.3-060503.202309130834_amd64.deb https://kernel.ubuntu.com/~kernel-ppa/mainline/v6.5.3/amd64/linux-headers-6.5.3-060503_6.5.3-060503.202309130834_all.deb https://kernel.ubuntu.com/~kernel-ppa/mainline/v6.5.3/amd64/linux-image-unsigned-6.5.3-060503-generic_6.5.3-060503.202309130834_amd64.deb https://kernel.ubuntu.com/~kernel-ppa/mainline/v6.5.3/amd64/linux-modules-6.5.3-060503-generic_6.5.3-060503.202309130834_amd64.deb
  # output:
  # [...]
  # Downloaded: 4 files, 190M in 2.5s (76.1 MB/s)
  dpkg -i *.deb
  # output:
  # Selecting previously unselected package linux-headers-6.5.3-060503.
  # [...]
  # Setting up linux-headers-6.5.3-060503 (6.5.3-060503.202309130834) ...
  # dpkg: dependency problems prevent configuration of linux-headers-6.5.3-060503-generic:
  #  linux-headers-6.5.3-060503-generic depends on libc6 (>= 2.38); however:
  #   Version of libc6:amd64 on system is 2.35-0ubuntu3.3.
  #
  # dpkg: error processing package linux-headers-6.5.3-060503-generic (--install):
  #  dependency problems - leaving unconfigured
  # Setting up linux-modules-6.5.3-060503-generic (6.5.3-060503.202309130834) ...
  # [...]
  # Errors were encountered while processing:
  #  linux-headers-6.5.3-060503-generic
  ```

  We're getting an error on setting up `linux-headers-6.5.3-060503-generic`, as it depends on `libc6 (>= 2.38)`, and we only have `2.35`. Not to worry (or should we?), we'll hack around this by editing `/var/lib/dpkg/status`, searching for `linux-headers-6.5.3-060503-generic`, replacing the `libc6 (>= 2.38)` with `libc6 (>= 2.35)` in `Depends:`, and then running:

  ```sh
  apt-get -qf install
  # output:
  # Reading package lists...
  # Building dependency tree...
  # Reading state information...
  # 0 upgraded, 0 newly installed, 0 to remove and 1 not upgraded.
  # 1 not fully installed or removed.
  # After this operation, 0 B of additional disk space will be used.
  # Setting up linux-headers-6.5.3-060503-generic (6.5.3-060503.202309130834) ...
  # needrestart is being skipped since dpkg has failed
  ```

- Hard-disable IPv6, from the kernel command line. This is not required in any way, just personal preference, so that I don't see IPv6 addresses when troubleshooting networking issues. IPv6 enabled might also cause the odd, obscure networking issue, like hostnames resolving to IPv6 IPs instead of IPv4.

  First we edit `/etc/default/grub` and append `ipv6.disable=1` to `GRUB_CMDLINE_LINUX_DEFAULT`. Then we edit `/etc/netplan/50-cloud-init.yaml` and remove all references to IPv6 addresses - `addresses`, `nameservers` and `routes` (otherwise `systemd` will block when configuring the network on next boot).

  Finally, run:

  ```sh
  update-grub
  # output:
  # Sourcing file `/etc/default/grub'
  # Sourcing file `/etc/default/grub.d/init-select.cfg'
  # Generating grub configuration file ...
  # Found linux image: /boot/vmlinuz-6.5.3-060503-generic
  # Found initrd image: /boot/initrd.img-6.5.3-060503-generic
  # Found linux image: /boot/vmlinuz-5.15.0-79-generic
  # Found initrd image: /boot/initrd.img-5.15.0-79-generic
  # Warning: os-prober will not be executed to detect other bootable partitions.
  # Systems on them will not be added to the GRUB boot configuration.
  # Check GRUB_DISABLE_OS_PROBER documentation entry.
  # done
  #
  # check grub config was updated:
  grep ipv6 /boot/grub/grub.cfg
  # output:
  #         linux   /boot/vmlinuz-6.5.3-060503-generic root=UUID=84613a9c-5cef-43c3-a3b3-48d3ef8843d6 ro  consoleblank=0 systemd.show_status=true console=tty1 console=ttyS0 ipv6.disable=1
  #                 linux   /boot/vmlinuz-6.5.3-060503-generic root=UUID=84613a9c-5cef-43c3-a3b3-48d3ef8843d6 ro  consoleblank=0 systemd.show_status=true console=tty1 console=ttyS0 ipv6.disable=1
  #                 linux   /boot/vmlinuz-5.15.0-79-generic root=UUID=84613a9c-5cef-43c3-a3b3-48d3ef8843d6 ro  consoleblank=0 systemd.show_status=true console=tty1 console=ttyS0 ipv6.disable=1
  ```

- Regardless of whether running Kubernetes or not, you should **always** run a time synchronization service on your servers! Ubuntu comes with `systemd-timesyncd` out of the box, but I've had issues with it in the past, suddenly stopping working, so I switched to `chrony`.

  ```sh
  apt-get -qy install chrony
  # [...]
  systemctl is-enabled chrony
  # enabled
  chronyc tracking
  # Reference ID    : 9F454532 (littlericket.me)
  # Stratum         : 3
  # Ref time (UTC)  : Sun Sep 17 20:40:32 2023
  # System time     : 0.001513181 seconds slow of NTP time
  # Last offset     : +0.000196304 seconds
  # RMS offset      : 0.001281233 seconds
  # Frequency       : 7.095 ppm slow
  # Residual freq   : -0.063 ppm
  # Skew            : 7.722 ppm
  # Root delay      : 0.034223251 seconds
  # Root dispersion : 0.001392885 seconds
  # Update interval : 65.3 seconds
  # Leap status     : Normal
  chronyc sources
  # MS Name/IP address         Stratum Poll Reach LastRx Last sample
  # ===============================================================================
  # ^+ prod-ntp-3.ntp1.ps5.cano>     2   6    37     5    -11ms[  -13ms] +/-   35ms
  # ^+ prod-ntp-4.ntp4.ps5.cano>     2   6    37     6  -7215us[-9417us] +/-   27ms
  # ^- prod-ntp-3.ntp4.ps5.cano>     2   6    37     4  +3221us[+3221us] +/-   21ms
  # ^- alphyn.canonical.com          2   6    37     4  +5813us[+3611us] +/-  100ms
  # ^* littlericket.me               2   6    37     4  +3149us[ +947us] +/-   18ms
  # ^+ fa.gnudb.org                  2   6    37     5  +3494us[+1292us] +/-   53ms
  # ^+ gamma.rueckgr.at              2   6    37     6  +3236us[+1034us] +/-   35ms
  # ^+ mail.light-speed.de           2   6    37     5  +5777us[+3575us] +/-   27ms
  ```

- Move the `ssh` server to a "random" port:

  ```sh
  cat <<EOT >/etc/ssh/sshd_config.d/01-port.conf
  Port 22322
  EOT
  ```

- Enable `bash` completion. For some reason, although the `bash-completion` package is installed, completion is not enabled. To enable, we edit `/etc/bash.bashrc` and uncomment the code block after the `# enable bash completion in interactive shells` line. We then restart the `bash` session and check if it works:

  ```sh
  exec bash -l
  systemctl status c<TAB><TAB>
  # chronyd.service              cloud-init-hotplugd.service  connman.service              cryptdisks-early.service
  # chrony.service               cloud-init-hotplugd.socket   console-getty.service        cryptdisks.service
  # cloud-config.service         cloud-init-local.service     console-screen.service       cryptsetup-pre.target
  # cloud-config.target          cloud-init.service           console-setup.service        cryptsetup.target
  # cloud-final.service          cloud-init.target            cron.service                 ctrl-alt-del.target
  ```

We can now reboot, and if we did everything right, the server should come back up with the new configuration:

```sh
systemctl reboot
# wait for server to come back up and ssh into it; don't forget we changed the port!
uname -a
# Linux test-cp-0 6.5.3-060503-generic #202309130834 SMP PREEMPT_DYNAMIC Wed Sep 13 08:42:33 UTC 2023 x86_64 x86_64 x86_64 GNU/Linux
ip add
# 1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
#     link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
#     inet 127.0.0.1/8 scope host lo
#        valid_lft forever preferred_lft forever
# 2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
#     link/ether 96:00:02:89:0d:9b brd ff:ff:ff:ff:ff:ff
#     altname enp0s3
#     inet 65.108.153.69/32 metric 100 scope global dynamic eth0
#        valid_lft 86282sec preferred_lft 86282sec
```
