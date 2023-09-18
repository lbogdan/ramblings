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

The only software prerequisite is a Linux distribution that you're familiar with, using `systemd`, a fairly recent kernel version, [cgroup v2](https://docs.kernel.org/admin-guide/cgroup-v2.html), and having packages for Kubernetes tools and components (`kubeadm`, `kubectl`, `kubelet`) and CRI runtime (either `containerd` or `cri-o`). I've been using Ubuntu since 12.04 or so (released more than 10 years ago), so it is my distribution of choice. For this I'll be using the current LTS release, Ubuntu 22.04.3.

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

  First we edit `/etc/default/grub` and append `ipv6.disable=1` to `GRUB_CMDLINE_LINUX_DEFAULT`. Then we edit `/etc/netplan/50-cloud-init.yaml` and remove (/ comment out) all references to IPv6 addresses - `addresses`, `nameservers` and `routes` (otherwise `systemd` will block when configuring the network on next boot).

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

Now this is out of the way, let's get to the cool stuff!

## Add Package Repositories

Before we can install the required packages, we first have to add the repositories they reside in. We'll add one for the Kubernetes tools and components (`kubeadm`, `kubectl`, `kubelet`), and two for the CRI runtime and its dependencies. We'll use `cri-o` for now, and to be a bit conservative, and not on the bleeding edge, like with the kernel, we'll install the previous version at the moment of writing this, `1.27`.

So, following the instructions from [Installing `kubeadm`](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/#installing-kubeadm-kubelet-and-kubectl) and [CRI-O Installation Instructions](https://github.com/cri-o/cri-o/blob/main/install.md#apt-based-operating-systems):

```sh
export KUBERNETES_MINOR="1.27"
export OS="xUbuntu_$(lsb_release -rs)"
curl -fsSL "https://pkgs.k8s.io/core:/stable:/v${KUBERNETES_MINOR}/deb/Release.key" | gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v${KUBERNETES_MINOR}/deb/ /" >/etc/apt/sources.list.d/kubernetes.list
curl -fsSL https://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable/${OS}/Release.key | gpg --dearmor -o /etc/apt/keyrings/libcontainers-archive-keyring.gpg
echo "deb [signed-by=/etc/apt/keyrings/libcontainers-archive-keyring.gpg] https://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable/${OS}/ /" >/etc/apt/sources.list.d/devel:kubic:libcontainers:stable.list
echo "deb [signed-by=/etc/apt/keyrings/libcontainers-archive-keyring.gpg] https://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable:/cri-o:/${KUBERNETES_MINOR}/${OS}/ /" >/etc/apt/sources.list.d/devel:kubic:libcontainers:stable:cri-o:${KUBERNETES_MINOR}.list
apt-get -q update
apt-cache policy kubeadm
# kubeadm:
#   Installed: (none)
#   Candidate: 1.27.6-1.1
#   Version table:
#      1.27.6-1.1 500
#         500 https://pkgs.k8s.io/core:/stable:/v1.27/deb  Packages
# [...]
apt-cache policy cri-o
# cri-o:
#   Installed: (none)
#   Candidate: 1.27.1~0
#   Version table:
#      1.27.1~0 500
#         500 https://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable:/cri-o:/1.27/xUbuntu_22.04  Packages
```

We can see that the latest Kubernetes `1.27` release is `1.27.6`. Interestingly, the official [release page](https://kubernetes.io/releases/#release-v1-27) shows "**Latest Release:** 1.27.5 (released: 2023-08-23)", but if we go to the [1.27 Changelog](https://github.com/kubernetes/kubernetes/blob/master/CHANGELOG/CHANGELOG-1.27.md), latest is indeed `1.27.6`.

As for `cri-o`, if we go to its [GitHub releases page](https://github.com/cri-o/cri-o/releases), we can see the latest `1.27` version is `1.27.1`.

> **Note**
> We're making this version checks because sometimes the package repositories lag a bit behind the actual releases, especially for `cri-o`. In out specific case, as the versions are not very recent, that's not a concern.

## Install Packages

It's time to finally install the required packages!

```sh
# make sure KUBERNETES_MINOR is still set
export KUBERNETES_VERSION="${KUBERNETES_MINOR}.6"
echo $KUBERNETES_VERSION 
# 1.27.6
apt-get -qy install "kubeadm=${KUBERNETES_VERSION}-1.1" "kubectl=${KUBERNETES_VERSION}-1.1" "kubelet=${KUBERNETES_VERSION}-1.1" cri-o runc
apt-mark hold kubeadm kubectl kubelet cri-o runc
```

We're also installing `runc`, which is the reference implementation of the [OCI runtime specification](https://github.com/opencontainers/runtime-spec), used by `cri-o` to actually manage containers' lifecycle, and "holding" the installed packages, which means they won't be automatically updated when running `apt-get upgrade`.

## CRI Runtime

Like I said above, `kubeadm` is the "official" Kubernetes tool to create a cluster, specifically its `kubeadm init` subcommand. If we take a look at its help (`kubeadm init --help`), we can see that, when invoked, it will execute through a list of phases, of which the first is "preflight - Run pre-flight checks". We can also execute a specific phase, so let's do a preflight check:

```sh
kubeadm init phase preflight | tee kubeadm-init-preflight.log
# I0918 09:21:28.063000   11270 version.go:256] remote version is much newer: v1.28.2; falling back to: stable-1.27
# [preflight] Running pre-flight checks
# error execution phase preflight: [preflight] Some fatal errors occurred:
#         [ERROR CRI]: container runtime is not running: output: time="2023-09-18T09:21:28Z" level=fatal msg="validate service connection: validate CRI v1 runtime API for endpoint \"unix:///var/run/containerd/containerd.sock\": rpc error: code = Unavailable desc = connection error: desc = \"transport: Error while dialing dial unix /var/run/containerd/containerd.sock: connect: no such file or directory\""
# , error: exit status 1
#         [ERROR FileContent--proc-sys-net-bridge-bridge-nf-call-iptables]: /proc/sys/net/bridge/bridge-nf-call-iptables does not exist
#         [ERROR FileContent--proc-sys-net-ipv4-ip_forward]: /proc/sys/net/ipv4/ip_forward contents are not set to 1
# [preflight] If you know what you are doing, you can make a check non-fatal with `--ignore-preflight-errors=...`
# To see the stack trace of this error execute with --v=5 or higher
```

So there's a few things here:

- by default, `kubeadm` uses the latest Kubernetes version, but because we're using `kubeadm` `1.27`, and the latest version is `1.28`, it falls back to the latest `1.27`; to be explicit, we'll use the `--kubernetes-version` flag in future invocations (**but**, for some weird reason, that only works in `kubeadm init`, and not in `kubeadm init phase`);

- the CRI runtime is not running; indeed, we installed it, but it's not set to start automatically;

- and we're missing a couple kernel configurations.

Let's first start the CRI runtime. But before that, we'll remove some default CNI network configurations, which the `cri-o` package ships with, but stand in the way more than help. We'll also create a `/var/lib/crio` folder, which `cri-o` uses for some bookkeeping, but for some weird reason it doesn't create:

```sh
rm -v /etc/cni/net.d/*.conflist
# removed '/etc/cni/net.d/100-crio-bridge.conflist'
# removed '/etc/cni/net.d/200-loopback.conflist'
mkdir /var/lib/crio
systemctl start crio
# Job for crio.service failed because the control process exited with error code.
# See "systemctl status crio.service" and "journalctl -xeu crio.service" for details.
```

Oops, it won't start. But why? Looking at the logs we see:

```sh
journalctl -u crio
# [...]
# Sep 18 09:40:17 test-cp-0 crio[11624]: time="2023-09-18 09:40:17.792619142Z" level=fatal msg="validating runtime config: runtime validation: invalid runtime_path for runtime 'runc': \"stat /usr/lib/cri-o-runc/sbin/runc: no such file or directory\""
# Sep 18 09:40:17 test-cp-0 systemd[1]: crio.service: Main process exited, code=exited, status=1/FAILURE
# Sep 18 09:40:17 test-cp-0 systemd[1]: crio.service: Failed with result 'exit-code'.
# Sep 18 09:40:17 test-cp-0 systemd[1]: Failed to start Container Runtime Interface for OCI (CRI-O).
which runc
# /usr/sbin/runc
```

Interesting, it's trying to use a `runc` from `/usr/lib/cri-o-runc/sbin/runc`, but our `runc` is in `/usr/sbin/runc`. Turns out `cri-o` comes with its own `runc` package, named `cri-o-runc`, but it's an older version, and that's why we used the Ubuntu's `runc` package:

```sh
apt-cache policy cri-o-runc
# cri-o-runc:
#   Installed: (none)
#   Candidate: 1.0.1~2
#   Version table:
#      1.0.1~2 500
#         500 https://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable/xUbuntu_22.04  Packages
apt-cache policy runc
# runc:
#   Installed: 1.1.7-0ubuntu1~22.04.1
#   Candidate: 1.1.7-0ubuntu1~22.04.1
#   Version table:
#  *** 1.1.7-0ubuntu1~22.04.1 500
#         500 https://mirror.hetzner.com/ubuntu/packages jammy-updates/main amd64 Packages
# [...]
```

This is configured in `/etc/crio/crio.conf.d/01-crio-runc.conf`, so let's make it right:

```sh
cat /etc/crio/crio.conf.d/01-crio-runc.conf 
# [crio.runtime.runtimes.runc]
# runtime_path = "/usr/lib/cri-o-runc/sbin/runc"
# runtime_type = "oci"
# runtime_root = "/run/runc"
cat <<EOT >/etc/crio/crio.conf.d/01-crio-runc.conf
[crio.runtime.runtimes.runc]
runtime_path = "/usr/sbin/runc"
runtime_type = "oci"
runtime_root = "/run/runc"
EOT
systemctl restart crio
systemctl status crio
# ● crio.service - Container Runtime Interface for OCI (CRI-O)
#      Loaded: loaded (/lib/systemd/system/crio.service; disabled; vendor preset: enabled)
#      Active: active (running) since Mon 2023-09-18 09:55:42 UTC; 3s ago
# [...]
# Sep 18 09:55:42 test-cp-0 systemd[1]: Started Container Runtime Interface for OCI (CRI-O).
#
# also start it at boot
systemctl enable crio
# Created symlink /etc/systemd/system/cri-o.service → /lib/systemd/system/crio.service.
# Created symlink /etc/systemd/system/multi-user.target.wants/crio.service → /lib/systemd/system/crio.service.
```

Nice, we got it running! We can use the `crictl` tool to get some insights into its state:

```sh
# crictl version
# Version:  0.1.0
# RuntimeName:  cri-o
# RuntimeVersion:  1.27.1
# RuntimeApiVersion:  v1
crictl info
# {
#   "status": {
#     "conditions": [
#       {
#         "type": "RuntimeReady",
#         "status": true,
#         "reason": "",
#         "message": ""
#       },
#       {
#         "type": "NetworkReady",
#         "status": false,
#         "reason": "NetworkPluginNotReady",
#         "message": "Network plugin returns error: No CNI configuration file in /etc/cni/net.d/. Has your network provider started?"
#       }
#     ]
#   },
#   "config": {
#     "sandboxImage": "registry.k8s.io/pause:3.9"
#   }
# }
crictl pods
# POD ID              CREATED             STATE               NAME                NAMESPACE           ATTEMPT             RUNTIME
crictl ps
# CONTAINER           IMAGE               CREATED             STATE               NAME                ATTEMPT             POD ID              POD
crictl images
# IMAGE               TAG                 IMAGE ID            SIZE
```

As expected, it doesn't have any images yet, and it's not running any pods / containers. But wait, why is the network not ready? Were we wrong in deleting the default CNI network configs? Well, no, Kubernetes [network plugins](https://kubernetes.io/docs/concepts/extend-kubernetes/compute-storage-net/network-plugins/) are supposed to be installed **after** you have a running cluster.

## Kernel Configuration

And so we only have two more kernel-related requirements left:

```sh
#         [ERROR FileContent--proc-sys-net-bridge-bridge-nf-call-iptables]: /proc/sys/net/bridge/bridge-nf-call-iptables does not exist
#         [ERROR FileContent--proc-sys-net-ipv4-ip_forward]: /proc/sys/net/ipv4/ip_forward contents are not set to 1
```

For the first, we need the `br_netfilter` kernel module loaded, and for the second we need to enable IP forwarding:

```sh
cat <<EOF >/etc/modules-load.d/kubeadm.conf
br_netfilter
EOF
cat <<EOF >/etc/sysctl.d/kubeadm.conf
net.ipv4.ip_forward                 = 1
EOF
# the above will effect at next boot, so we need to explicitly enable them:
modprobe br_netfilter
sysctl --system 2>/dev/null | grep ip_forward
# net.ipv4.ip_forward = 1
```

Feeling lucky? Let's run a preflight check again:

```sh
# this will take a while, depending on your network speed,
# as it's pulling the Kubernetes components' images
kubeadm init phase preflight | tee -a kubeadm-init-preflight.log
# I0918 10:19:28.375256   13508 version.go:256] remote version is much newer: v1.28.2; falling back to: stable-1.27
# [preflight] Running pre-flight checks
# [preflight] Pulling images required for setting up a Kubernetes cluster
# [preflight] This might take a minute or two, depending on the speed of your internet connection
# [preflight] You can also perform this action in beforehand using 'kubeadm config images pull'
crictl images
# IMAGE                                     TAG                 IMAGE ID            SIZE
# registry.k8s.io/coredns/coredns           v1.10.1             ead0a4a53df89       53.6MB
# registry.k8s.io/etcd                      3.5.7-0             86b6af7dd652c       297MB
# registry.k8s.io/kube-apiserver            v1.27.6             19b9246d37c8b       122MB
# registry.k8s.io/kube-controller-manager   v1.27.6             7810e6aed9778       114MB
# registry.k8s.io/kube-proxy                v1.27.6             ec57bbfaaae73       72.7MB
# registry.k8s.io/kube-scheduler            v1.27.6             993d768b9b96e       59.8MB
# registry.k8s.io/pause                     3.9                 e6f1816883972       750kB
```

Could this be? Are we finally ready to create the cluster? Let's see!

## Finally, a Cluster!

There's one last detail you need to decide on before creating the cluster: the pod network CIDR. From `kubeadm init`'s help:

```sh
kubeadm init --help | grep cidr
      # --pod-network-cidr string              Specify range of IP addresses for the pod network. If set, the control plane will automatically allocate CIDRs for every node.
      # --service-cidr string                  Use alternative range of IP address for service VIPs. (default "10.96.0.0/12")
```

Actually, there's two network CIDRs, for both pods and services, but we'll use the default for services. And for the pod network we'll use `192.168.0.0/16`, which is Calico's default, the network plugin we'll be using.

> **Warning**
> When you decide on which networks to use, make sure they don't overlap with any other networks you want to reach from the pods running inside the cluster, or networks already in use on your cluster nodes.

```sh
# make sure KUBERNETES_VERSION is still set
kubeadm init --kubernetes-version "$KUBERNETES_VERSION" --pod-network-cidr 192.168.0.0/16 | tee kubeadm-init.log
# [init] Using Kubernetes version: v1.27.6
# [preflight] Running pre-flight checks
# [preflight] Pulling images required for setting up a Kubernetes cluster
# [preflight] This might take a minute or two, depending on the speed of your internet connection
# [preflight] You can also perform this action in beforehand using 'kubeadm config images pull'
# [certs] Using certificateDir folder "/etc/kubernetes/pki"
# [certs] Generating "ca" certificate and key
# [certs] Generating "apiserver" certificate and key
# [certs] apiserver serving cert is signed for DNS names [kubernetes kubernetes.default kubernetes.default.svc kubernetes.default.svc.cluster.local test-cp-0] and IPs [10.96.0.1 65.108.153.69]
# [certs] Generating "apiserver-kubelet-client" certificate and key
# [certs] Generating "front-proxy-ca" certificate and key
# [certs] Generating "front-proxy-client" certificate and key
# [certs] Generating "etcd/ca" certificate and key
# [certs] Generating "etcd/server" certificate and key
# [certs] etcd/server serving cert is signed for DNS names [localhost test-cp-0] and IPs [65.108.153.69 127.0.0.1 ::1]
# [certs] Generating "etcd/peer" certificate and key
# [certs] etcd/peer serving cert is signed for DNS names [localhost test-cp-0] and IPs [65.108.153.69 127.0.0.1 ::1]
# [certs] Generating "etcd/healthcheck-client" certificate and key
# [certs] Generating "apiserver-etcd-client" certificate and key
# [certs] Generating "sa" key and public key
# [kubeconfig] Using kubeconfig folder "/etc/kubernetes"
# [kubeconfig] Writing "admin.conf" kubeconfig file
# [kubeconfig] Writing "kubelet.conf" kubeconfig file
# [kubeconfig] Writing "controller-manager.conf" kubeconfig file
# [kubeconfig] Writing "scheduler.conf" kubeconfig file
# [kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
# [kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
# [kubelet-start] Starting the kubelet
# [control-plane] Using manifest folder "/etc/kubernetes/manifests"
# [control-plane] Creating static Pod manifest for "kube-apiserver"
# [control-plane] Creating static Pod manifest for "kube-controller-manager"
# [control-plane] Creating static Pod manifest for "kube-scheduler"
# [etcd] Creating static Pod manifest for local etcd in "/etc/kubernetes/manifests"
# [wait-control-plane] Waiting for the kubelet to boot up the control plane as static Pods from directory "/etc/kubernetes/manifests". This can take up to 4m0s
# [apiclient] All control plane components are healthy after 17.003738 seconds
# [upload-config] Storing the configuration used in ConfigMap "kubeadm-config" in the "kube-system" Namespace
# [kubelet] Creating a ConfigMap "kubelet-config" in namespace kube-system with the configuration for the kubelets in the cluster
# [upload-certs] Skipping phase. Please see --upload-certs
# [mark-control-plane] Marking the node test-cp-0 as control-plane by adding the labels: [node-role.kubernetes.io/control-plane node.kubernetes.io/exclude-from-external-load-balancers]
# [mark-control-plane] Marking the node test-cp-0 as control-plane by adding the taints [node-role.kubernetes.io/control-plane:NoSchedule]
# [bootstrap-token] Using token: yx2bhq.p1hvj3eopm4xurtq
# [bootstrap-token] Configuring bootstrap tokens, cluster-info ConfigMap, RBAC Roles
# [bootstrap-token] Configured RBAC rules to allow Node Bootstrap tokens to get nodes
# [bootstrap-token] Configured RBAC rules to allow Node Bootstrap tokens to post CSRs in order for nodes to get long term certificate credentials
# [bootstrap-token] Configured RBAC rules to allow the csrapprover controller automatically approve CSRs from a Node Bootstrap Token
# [bootstrap-token] Configured RBAC rules to allow certificate rotation for all node client certificates in the cluster
# [bootstrap-token] Creating the "cluster-info" ConfigMap in the "kube-public" namespace
# [kubelet-finalize] Updating "/etc/kubernetes/kubelet.conf" to point to a rotatable kubelet client certificate and key
# [addons] Applied essential addon: CoreDNS
# [addons] Applied essential addon: kube-proxy

# Your Kubernetes control-plane has initialized successfully!

# To start using your cluster, you need to run the following as a regular user:

#   mkdir -p $HOME/.kube
#   sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
#   sudo chown $(id -u):$(id -g) $HOME/.kube/config

# Alternatively, if you are the root user, you can run:

#   export KUBECONFIG=/etc/kubernetes/admin.conf

# You should now deploy a pod network to the cluster.
# Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
#   https://kubernetes.io/docs/concepts/cluster-administration/addons/

# Then you can join any number of worker nodes by running the following on each as root:

# kubeadm join 65.108.153.69:6443 --token yx2bhq.p1hvj3eopm4xurtq \
#         --discovery-token-ca-cert-hash sha256:a566acdc5d026db191b7ccc04da9a8b5331414846c8a5de072c53b337706ec9d
export KUBECONFIG=/etc/kubernetes/admin.conf
kubectl get nodes
# NAME        STATUS     ROLES           AGE   VERSION
# test-cp-0   NotReady   control-plane   87s   v1.27.6
```

Hooray, we have a functional clus... Wait, why is the node `NotReady`?!

```sh
kubectl describe node test-cp-0
# Conditions:
#   Type             Status  LastHeartbeatTime                 LastTransitionTime                Reason                       Message
#   ----             ------  -----------------                 ------------------                ------                       -------
# [...]
#   Ready            False   Mon, 18 Sep 2023 10:38:27 +0000   Mon, 18 Sep 2023 10:38:21 +0000   KubeletNotReady              container runtime network not ready: NetworkReady=false reason:NetworkPluginNotReady message:Network plugin returns error: No CNI configuration file in /etc/cni/net.d/. Has your network provider started?
```

Oh, of course, we don't have a network plugin yet! This also affects (some of) the pods starting:

```sh
kubectl get pods -A -o wide
# NAMESPACE     NAME                                READY   STATUS    RESTARTS   AGE     IP              NODE        NOMINATED NODE   READINESS GATES
# kube-system   coredns-5d78c9869d-nshvg            0/1     Pending   0          3m59s   <none>          <none>      <none>           <none>
# kube-system   coredns-5d78c9869d-txn4x            0/1     Pending   0          3m59s   <none>          <none>      <none>           <none>
# kube-system   etcd-test-cp-0                      1/1     Running   0          4m13s   65.108.153.69   test-cp-0   <none>           <none>
# kube-system   kube-apiserver-test-cp-0            1/1     Running   0          4m13s   65.108.153.69   test-cp-0   <none>           <none>
# kube-system   kube-controller-manager-test-cp-0   1/1     Running   0          4m13s   65.108.153.69   test-cp-0   <none>           <none>
# kube-system   kube-proxy-74f6z                    1/1     Running   0          3m59s   65.108.153.69   test-cp-0   <none>           <none>
# kube-system   kube-scheduler-test-cp-0            1/1     Running   0          4m13s   65.108.153.69   test-cp-0   <none>           <none>
kubectl -n kube-system describe pod coredns-5d78c9869d-nshvg
# [...]
# Events:
#   Type     Reason            Age   From               Message
#   ----     ------            ----  ----               -------
#   Warning  FailedScheduling  5m5s  default-scheduler  0/1 nodes are available: 1 node(s) had untolerated taint {node.kubernetes.io/not-ready: }. preemption: 0/1 nodes are available: 1 Preemption is not helpful for scheduling..
```

But why are all the other pods `Running`? Good question, it's because they use host networking, so they don't need a network plugin to start, e.g.:

```sh
kubectl -n kube-system get pod kube-proxy-74f6z -o yaml | grep hostNetwork
#   hostNetwork: true
```

## Network Plugin: Calico

Alrighty, then, it looks like installing a network plugin will finally get us a functional cluster! As I said above, we'll be using Calico. Let's follow their [install instructions](https://docs.tigera.io/calico/latest/getting-started/kubernetes/self-managed-onprem/onpremises#install-calico-with-kubernetes-api-datastore-50-nodes-or-less) for a (small) self-managed on-premise cluster:

```sh
curl -O https://raw.githubusercontent.com/projectcalico/calico/v3.26.1/manifests/calico.yaml
# take a quick look at the manifest
less calico.yaml
# and then apply it; make sure KUBECONFIG is still set
kubectl apply -f calico.yaml
# serviceaccount/calico-kube-controllers created
# [...]
# deployment.apps/calico-kube-controllers created
```

This creates all required Kubernetes objects to instruct our pristine cluster how to run the network plugin. It will pull some container images, so depending on your network speed, it will take a little while to converge. After a few tens of second to a minute, we should see this:

```sh
kubectl get no -o wide
# NAME        STATUS   ROLES           AGE   VERSION   INTERNAL-IP     EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION         CONTAINER-RUNTIME
# test-cp-0   Ready    control-plane   21m   v1.27.6   65.108.153.69   <none>        Ubuntu 22.04.3 LTS   6.5.3-060503-generic   cri-o://1.27.1
kubectl get pods -A -o wide
# NAMESPACE     NAME                                       READY   STATUS    RESTARTS   AGE     IP              NODE        NOMINATED NODE   READINESS GATES
# kube-system   calico-kube-controllers-85578c44bf-lzmrz   1/1     Running   0          4m35s   192.168.0.67    test-cp-0   <none>           <none>
# kube-system   calico-node-brsmd                          1/1     Running   0          4m35s   65.108.153.69   test-cp-0   <none>           <none>
# kube-system   coredns-5d78c9869d-nshvg                   1/1     Running   0          22m     192.168.0.66    test-cp-0   <none>           <none>
# kube-system   coredns-5d78c9869d-txn4x                   1/1     Running   0          22m     192.168.0.65    test-cp-0   <none>           <none>
# kube-system   etcd-test-cp-0                             1/1     Running   0          22m     65.108.153.69   test-cp-0   <none>           <none>
# kube-system   kube-apiserver-test-cp-0                   1/1     Running   0          22m     65.108.153.69   test-cp-0   <none>           <none>
# kube-system   kube-controller-manager-test-cp-0          1/1     Running   0          22m     65.108.153.69   test-cp-0   <none>           <none>
# kube-system   kube-proxy-74f6z                           1/1     Running   0          22m     65.108.153.69   test-cp-0   <none>           <none>
# kube-system   kube-scheduler-test-cp-0                   1/1     Running   0          22m     65.108.153.69   test-cp-0   <none>           <none>
```

The node is `Ready`, the `coredns` pods are `Running`, and we have two more `calico` pods, created by the applied manifest.

> **Note**
> As an aside, I think that's one of the Kubernetes' strengths: you define a manifest following a well-defined API, and after a little while, you have your cluster converging, configuring and running whatever workloads you defined in your manifest. Just imagine doing the same to orchestrate, configure and run workloads on multiple nodes with assorted tools and / or bespoke scripts!

## Cluster Validation Checks

Now that we have a functional cluster, let's run a simple workload and check if we can access it. We'll use the official [`nginx` image](https://hub.docker.com/_/nginx):

```sh
kubectl run nginx --image nginx:1.25.2
# pod/nginx created
root@test-cp-0:~# kubectl get pods
# NAME    READY   STATUS    RESTARTS   AGE
# nginx   0/1     Pending   0          6s
```

> **Warning**
> As best practice, **never** use image specs without a version tag, or with the `latest` tag (they are equivalent). `latest` follows whatever version is the latest, and it will automatically upgrade to minor, and even major versions whenever your pod restarts.

Hm, it's `Pending`, let's check why:

```sh
kubectl run nginx --image nginx:1.25.2
# [...]
# Events:
#   Type     Reason            Age   From               Message
#   ----     ------            ----  ----               -------
#   Warning  FailedScheduling  78s   default-scheduler  0/1 nodes are available: 1 node(s) had untolerated taint {node-role.kubernetes.io/control-plane: }. preemption: 0/1 nodes are available: 1 Preemption is not helpful for scheduling..
```

By default, `kubeadm` taints the control-plane nodes with the `node-role.kubernetes.io/control-plane` taint, so that no "regular" workload will be scheduled on them, and that's what we see with our `nginx` pod. As we have a one-node cluster for now, on which we want to run all kinds of workloads, we have to remove the taint:

```sh
kubectl taint node test-cp-0 node-role.kubernetes.io/control-plane:NoSchedule-
# node/test-cp-0 untainted
kubectl get pods -o wide
# NAME    READY   STATUS    RESTARTS   AGE     IP             NODE        NOMINATED NODE   READINESS GATES
# nginx   1/1     Running   0          6m29s   192.168.0.68   test-cp-0   <none>           <none>
```

Cool, our pod is running, and got allocated the `192.168.0.68` IP. Let's check if we can access it, first from the node:

```sh
# 192.168.0.68 is the pod IP shown above
curl -i http://192.168.0.68/
# HTTP/1.1 200 OK
# Server: nginx/1.25.2
# Date: Mon, 18 Sep 2023 11:25:52 GMT
# Content-Type: text/html
# Content-Length: 615
# Last-Modified: Tue, 15 Aug 2023 17:03:04 GMT
# Connection: keep-alive
# ETag: "64dbafc8-267"
# Accept-Ranges: bytes

# <!DOCTYPE html>
# <html>
# <head>
# <title>Welcome to nginx!</title>
# [...]
```

It works! One last check, let's expose it using a `NodePort` service and access it from the Internet:

```sh
kubectl expose pod nginx --port 80 --type NodePort
# service/nginx exposed
kubectl get services -o wide
# NAME    TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
# nginx   NodePort   10.102.38.250   <none>        80:31557/TCP   30s
```

The pod should now be publicly exposed on all nodes, on the `31557/TCP` port. Let's check that we can access it from our local machine:

```sh
# 65.108.153.69 is our node public IP, 31557 is the service node port
curl -i http://65.108.153.69:31557/
# HTTP/1.1 200 OK
# Server: nginx/1.25.2
# Date: Mon, 18 Sep 2023 11:34:12 GMT
# Content-Type: text/html
# Content-Length: 615
# Last-Modified: Tue, 15 Aug 2023 17:03:04 GMT
# Connection: keep-alive
# ETag: "64dbafc8-267"
# Accept-Ranges: bytes

# <!DOCTYPE html>
# <html>
# <head>
# <title>Welcome to nginx!</title>
# [...]
#
# or load http://65.108.153.69:31557/ in a browser
#
# now let's cleanup after ourselves:
kubectl delete service nginx
# service "nginx" deleted
# the pod delete takes ~30s, as nginx doesn't handle SIGTERM
# see https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/
kubectl delete pod nginx
# pod "nginx" deleted
```

Victory! We now finally have an (almost) functional cluster! We still need some persistent storage, an ingress controller, a way to generate TLS certificates for exposed workloads, and so on, and we'll explore all these in a future article.

There's also some questionable `kubeadm` defaults, which are not immediately apparent, so let's explore them in the next article!
