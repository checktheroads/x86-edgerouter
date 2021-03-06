# Hardware

Supermicro `5019D-FN8TP` with default onboard network interfaces, which are:
* 4x Intel 1 Gbps I350 (1000BASE-T)
* 2x Intel 10 Gbps X722 (10GBASE-T)
* 2x Intel 10 Gbps X722 (10GbE SFP)

---

# Intel Driver
## Reference
* [Ubuntu/Debain Intel Driver Installation Guide](http://ask.xmodulo.com/download-install-ixgbe-driver-ubuntu-debian.html)
* ["`code model kernel does not support PIC mode`" Issue](https://github.com/lwfinger/rtlwifi_new/issues/390#issuecomment-433706382)

## Initial Attempt
I attempted to just #yolo and basically boot the machine up, and start with the FD.io VPP quickstart guide. Which worked, but the 10G interfaces didn't get imported into VPP, but *did* get yanked from the Linux kernel. So for a while I was in a funny state of not being able to see the interfaces via the kernel (`ip link show`), nor could I see them in VPP (`vppctl show interfaces`). After digging through logs it seemed that VPP wasn't able to address the interfaces due to not having drivers installed. Ultimately I uninstalled `vpp*` and `dpdk*`, rebooted, and the interfaces came back.

## Intel Driver Installation
I first attempted to install the drivers for the X722 controllers using the posted drivers on [Intel.com](https://downloadcenter.intel.com/download/22283/Intel-Ethernet-Adapter-Complete-Driver-Pack), but I was never able to `insmod ./i40e.ko` due to a complaint stating `insmod: error inserting './i40e.ko': Invalid module format.`. It seemed that the version on Intel.com (`i40e-2.7.27`) was specific to Linux kernel version `4.15.0-29-generic`, and Ubuntu 18.04 bionic is `4.15.0-**43**-generic` (eye roll). After digging through the README of the incorrect driver, I found:

```text
Due to the continuous development of the Linux kernel, the drivers are updated
 more often than the bundled releases. The latest driver can be found on
 http://e1000.sourceforge.net (and also on http://downloadcenter.intel.com.)
```

So, I checked the [SourceForge repo](https://sourceforge.net/projects/e1000/) and sure enough, there's a `2.7.29` version located [here](https://sourceforge.net/projects/e1000/files/i40e%20stable/2.7.29/).

After downloading this and following the README instructions to install, slighly modified by [this nice little guide](http://ask.xmodulo.com/download-install-ixgbe-driver-ubuntu-debian.html), I was able to successfully install the drivers. Brief steps below:

```console
# tar zxf i40e-2.7.29.tar.gz
# cd i40e-2.7.29/src/
# make install
# insmod ./i40e.ko
# modprobe i40e
```

# Vector Packet Processing (VPP)
## Reference
* [VPP Wiki](https://wiki.fd.io/view/VPP)
* [FD.io Quick Start Guide](https://docs.google.com/document/d/1zqYN7qMavgbdkPWIJIrsPXlxNOZ_GhEveHQxpYr3qrg/edit?usp=sharing)

## Installation

<!--Even though I'm running Ubuntu 18.04 (bionic), I did in fact add the 16.04 (xenial) repo per the quick start instructions:

```console
# cat /etc/apt/sources.list.d/99fd.io.list
deb [trusted=yes] https://nexus.fd.io/content/repositories/fd.io.ubuntu.xenial.main/ ./
```

I first attempted to, again, #yolo and manually change this to the bionic version of the repo, but this never worked. After following the instructions like a good little student, I was able to `apt update` and actually see the `vpp` packages.

For Ubuntu 18.04 (bionic beaver), add the relevant repo:

```console
# echo "deb [trusted=yes] https://packagecloud.io/fdio/master/ubuntu/ bionic main" > /etc/apt/sources.list.d/99fd.io.list
```

Update the sources, and install the packages:

```console
# apt update
# apt -y install vpp-lib vpp vpp-plugins
```-->
It'd be great if we could follow the FD.io quick start guide and simply `apt install -y vpp-lib vpp vpp-plugins` and move on, but because we need to use some special VPP plugins, we have to compile VPP from the source and install manually. Sad face.

Clone the `vpp` and `vppsb` (sandbox) repos:

```console
# git clone https://gerrit.fd.io/r/vpp /opt/vpp
# git clone https://gerrit.fd.io/r/vppsb /opt/vppsb
```

Begin the build process:

```console
# cd /opt/vpp
# make install-dep; make bootstrap; make build
```

Add this bit to the `/opt/vppsb/router/router/tap_inject_node.c` file:

```console
# nano /opt/vppsb/router/router/tap_inject_node.c
```

```c
#include <sys/uio.h>
```

Add these symlinks:

```console
# ln -sf /opt/vppsb/netlink
# ln -sf /opt/vppsb/router
# ln -sf /opt/vppsb/netlink/netlink.mk build-data/packages/
# ln -sf /opt/vppsb/router/router.mk build-data/packages/
```

Build VPP:

```console
# cd /opt/vpp/build-root
# sh ./vagrant/build.sh
# make V=0 PLATFORM=vpp TAG=vpp_debug netlink-install router-install
```

Install the packages:

```console
# dpkg -i *.deb
# cp -p /opt/vpp/build-root/install-vpp_debug-native/router/lib64/router.so.0.0.0 /usr/lib/vpp_plugins/router.so
```
In my case, I was never able to install `vpp-plugins` due to it complaining about `libssl`:

```text
Depends: libssl1.0.0 (>= 1.0.0) but it is not installable
```

...even though I had it installed:

```console
# apt list | grep libssl1.
libssl1.0.0/bionic-updates,bionic-security,now 1.0.2n-1ubuntu5.2 amd64 [installed]
libssl1.1/bionic-updates,bionic-security,now 1.1.0g-2ubuntu4.3 amd64 [installed]
```

So, after some reseach, I installed the following package manually and was able to install `vpp-plugins`

```console
# wget http://security.ubuntu.com/ubuntu/pool/universe/m/mbedtls/libmbedcrypto0_2.2.1-2ubuntu0.2_amd64.deb
# dpkg -i libmbedcrypto0_2.2.1-2ubuntu0.2_amd64.deb
# dpkg -i vpp-plugins*.deb
```

Finally, restart the `vpp` service:

```console
# service vpp restart
```
### Driver woes

I fought for a while to get VPP to recognize the Intel i40e onboard NICs on the Supermicro `5019D-FN8TP`. Any time VPP started I saw `dpdk: Unsupported PCI device 0x8086:0x37c8 found at PCI address 0000:b6:00.0` in the logs (`journalctl -xe`). After digging through the documentation and the bowels of Google, I tried manually whitelisting (and renaming) the interfaces, and that did the trick. Here's the bit I added to `/etc/vpp/startup.conf` under the `dpdk` section:

```
dev 0000:65:00.1 {
  name ge1
}
dev 0000:65:00.2 {
  name ge2
}
dev 0000:65:00.3 {
  name ge3
}
dev 0000:b7:00.0 {
  name te0
}
dev 0000:b7:00.1 {
  name te1
}
dev 0000:b7:00.2 {
  name te2
}
dev 0000:b7:00.3 {
  name te3
}
```

I've placed the full `startup.conf` in the [repo](https://github.com/checktheroads/x86-edgerouter/tree/master/docs/startup.conf) for reference.

Additionally, the Intel i40e driver requires the vfio_pci driver to be loaded. You can load it once with:

```console
# modprobe vfio_pci
```

Or ensure it always loads at startup:

```console
# echo "vfio_pci" >> /etc/modules
```

## Configuration

Enable tap-inject:

```console
# vppctl enable tap-inject
```

And verify the physical interface to tap interface mappings:

```console
# vppctl show tap-inject
```
