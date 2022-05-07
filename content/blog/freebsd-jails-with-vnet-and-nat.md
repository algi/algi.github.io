---
title: "FreeBSD jails with VNET and NAT"
date: 2022-05-07
description: "How to create FreeBSD jail from scratch using VNET and NAT."
slug: "freebsd-jails-with-vnet-and-nat"
tags: [ "freebsd", "vnet", "jails", "nat", "pf" ]
categories: [ "FreeBSD" ]
---

Since my early days with FreeBSD I have been fascinated by jails and their ability to form their own network stack. With the help of VNET, jails can even define their own firewall rules (if allowed), use DHCP to get an IP address, etc. Here I'm presenting a unique design, which I haven't found anywhere else. It is thoroughly documented with all steps required. It can serve various use cases: virtualised VM deployed in cloud that has only one network interface and one IP address. Or even running locally on a laptop.

The network setup I present here is based on `if_bridge(4)` network interface with a help of `jib` (Jail-Interface-Bridge) tool to make the scripting easier. NAT provides the network translation between bridge and the physical interface. Virtual network stack (VNET) is then provided to the jail using an `epair(4)` interface, which closely resembles an Ethernet cable with two ends. One end is connected to the bridge and the other one to the jail.

The following graph illustrates my network stack I'm using in this article: 

```
vnet <---> vnet0bridge (10.192.0.1/24) <---> e0a_jamulus <---> e0b_jamulus (10.192.0.10)
```

Components are following:

* `vnet` - physical interface
* `vnet0bridge` - cloned interface, bridge
* `e0a_jamulus` - epair end connected to the bridge
* `e0b_jamulus` - epair end connected to the jail

# Creating VNET jail with NAT step-by-step
## Bridge Interface
The bridge interface is created during boot process using `/etc/rc.conf` file directives. Its name must conform to the naming convention that `jib` script expects - `<ext_if><bridge_if>`. Because the bridge will serve as default router, it must have its own IP address assigned. The following example shows how to perform the configuration: 

```
cloned_interfaces="bridge0"                # bridge for jails
ifconfig_bridge0_name="vtnet0bridge"       # renamed interface
ifconfig_vtnet0bridge="inet 10.192.0.1/24" # IP address of the bridge interface
```
Note that my physical interface is called `vnet0`. Let's move to the network translation.

## NAT
I must admit I fell in love with `pf` firewall. The configuration is well documented and easy to follow. I use macros to define the interfaces. NAT is provided for the whole internal network, whose address is taken from the bridge interface. The last line illustrates how to redirect requests from the host to the jail's IP address. The following example is part of `/etc/pf.conf` file:

```
ext_if="vtnet0"       # physical interface
int_if="vtnet0bridge" # bridge interface

nat pass on $ext_if from $int_if:network to any -> ($ext_if)
rdr pass on $ext_if inet proto udp from any to $ext_if port 22124 -> 10.192.0.10
```

With this NAT configuration, the jail can now access the internet and can be reached from the outside as well. Time to create the jail!

## Jail Configuration
The jail is configured via `/etc/jail.conf` file. The `jib` script was installed by running `install /usr/share/examples/jail/jib /usr/local/bin`. It's also necessary to make it executable.

The following example explains the configuration itself. `jib` is heavily utilizing various naming conventions, so the names of the actual interfaces might not be completely obvious. I'm using comments to explain which alias stands for which interface. The pre-start script creates epair interfaces and connects them to the bridge and the jail. The post-stop script is responsible for removing the interfaces created before. `vnet.interface` specifies the name of the epair interface that is exposed to the jail. VNET automatically enables the use of `ping` utility, so there's no need to enable it explicitly. The following example shows the content of `/etc/jail.conf` file:

```
jamulus {
        host.hostname = "jamulus.beastie.local";    # hostname
        path = "/jail/jamulus";                     # root directory
        exec.clean;                                 # clean environment variables
        mount.devfs;                                # mount devfs

        vnet;                                       # requires bridge called vtnet0bridge with assigned IP address
        vnet.interface = "e0b_jamulus";             # name of epair used inside the jail

        exec.prestart += "jib addm jamulus vtnet0"; # jamulus=e0a_jamulus (external epair), vtnet0=vtnet0bridge (bridge)
        exec.poststop += "jib destroy jamulus";

        exec.start += "/bin/sh /etc/rc";
        exec.stop = "/bin/sh /etc/rc.shutdown jail";

        exec.consolelog = "/var/log/jail_jamulus_console.log";
}
```

## Jail Networking
The last step is to configure networking in the jail itself. The default router is our bridge interface, so we need to set its IP address there. The other part of epair interface is visible only in the jail, so we set the jail's IP address here. The following example shows jail's `/etc/rc.conf` file:

```
defaultrouter="10.192.0.1"
ifconfig_e0b_jamulus="10.192.0.10"
```

It can be also useful to disable `sendmail` during testing, so it doesn't slow down the start up process. This can be done by setting `sendmail_enable="NONE" in the rc file.

## Final touches
The jail is now ready to be used, congratulations! In order to start it automatically during boot add following lines to the host's `/etc/rc.conf` file:

```
jail_enable="YES"
jail_list="jamulus"
```

The jail is started using `service jail start` command. It's possible to finally install the required packages using command `pkg -j jamulus install <package>` from the host. We can "jump in" using `jexec jamulus sh` command. Enjoy!

# Resources
This blog post wouldn't be possible without the help of the following blog posts, and books from FreeBSD guru Michael W Lucas.

## Blogs
* [FreeBSD Jail with Single IP](http://kbeezie.com/freebsd-jail-single-ip/)
* [FreeBSD jails and vnet from scratch](https://www.amoradi.org/20210908201936.html)
* [FreeBSD Jails and Networking](https://etherealwake.com/2021/08/freebsd-jail-networking/)
* [MasonLoringBliss / JailsEpair - FreeBSD wiki](https://wiki.freebsd.org/MasonLoringBliss/JailsEpair)
* [Jails vnet - FreeBSD Mastery - multiple interfaces](https://forums.freebsd.org/threads/jails-vnet-freebsd-mastery-multiple-interfaces.70356/)

## Books
* [FreeBSD Mastery: Jails - Michael W Lucas](https://www.tiltedwindmillpress.com/product/fmjail/)
* [Absolute FreeBSD - Michael W Lucas](https://nostarch.com/absfreebsd3)

