---
title: "Docker in FreeBSD 13"
date: 2021-12-25
categories:
- FreeBSD
tags:
- bhyve
- docker
- freebsd
---

FreeBSD 13 does not (yet) contain working Docker machine, although several implementations are on the way. This post explains how to install Docker daemon in Alpine Linux running in Bhyve. It also shows how to create NAT in Bhyve. I have to use NAT, because I use link aggregation (failover) between Wi-Fi and Ethernet.

I will use following tools:

* `pf` - Packet Filter, part of the base system
* `vm-bhyve, grub2-bhyve` - CLI front end for Bhyve
* `tmux` - optional, replacement for default `cu`

## Networking
### System configuration
Let's start with setting up the system. We need to enable gateway and packet filter. Normally I prefer `doasedit vim /etc/rc.conf`, because I like to organise content of the file and also add notes. Here I'm using `sysrc` for simplicity:

```
# sysrc gateway_enabled=YES
# sysrc pf_enabled=YES
```

Second step is to enable IP forwarding. You can achieve that by adding a following line to `/etc/sysctl.conf`:

```
net.inet.ip.forwarding=1
```

The easiest thing to enable all these settings is to reboot the system. Experienced sysadmins might tell you that it's not necessary and that you could simply enable the service on the fly and run `sysctl`. Because I'm not a sysadmin, I cannot really recommend that with confidence, sorry.

### Packet Filter
I will skip rules configuration and focus only on creating NAT. If you are a novice and you run this on your laptop/desktop, I would probably recommend that you close all ports and allow all requests out. But please do read PF documentation and think about your use case first.

In this example, I choose to use `10.0.1.0/24` range for my VMs. You are welcome to use whatever range you wish, or even use IPv6 if you are into that sort of thing (no judgement). The configuration is very simple, you just add one line to `/etc/pf.conf`:

```
nat on lagg0 from {10.0.1.0/24} to any -> (lagg0)
```

You can parametrise the network interface to avoid repetition. Verify the rules and reload PF.

```
# pfctl -nvf /etc/pf.conf
# pfctl -F all -f /etc/pf.conf
```

*Note: Here I flush everything, but I have a strong feeling that this is an overkill. Restarting the service would also be an option.*

## Bhyve
### Installation and Configuration
If you have never used Bhyve, I recommend that you follow steps 1-6 from the [vm-bhyve](https://github.com/churchers/vm-bhyve) Github page. I will not repeate the steps here, because they might change in the future. Also remember to install both packages: `vm-bhyve` and `grub2-bhyve`. Linux requires Grub boot loader.

It's also a good idea to install `tmux` and follow [instructions](https://github.com/churchers/vm-bhyve/wiki/Using-tmux
) on the wiki to configure it. I find `tmux` much more convenient to use than default `cu`.

### Switch Interface
With having `vm-bhyve` installed and configured, it's time to create bridge for our VM. Bhyve creates it's own bridge interface, but we need to also tell it about our NAT. Create a new switch and verify that everything is correct:

```
# vm switch create -a 10.0.1.1/24 public  
# vm switch list
NAME    TYPE      IFACE      ADDRESS      PRIVATE  MTU  VLAN  PORTS
public  standard  vm-public  10.0.1.1/24  no       -    -     -
```

I follow the convention and call the switch `public`. You can see that `vm-bhyve` also created interface for you. Also notice that `10.0.1.1` is our gateway.

### Download ISO
It's time to download the ISO from [alpinelinux.org](https://www.alpinelinux.org/downloads/). I choose to download Virtual image distribution, since I don't need any extra drivers. The ISO is quite small, only 52M. Once you have it downloaded, move it to your ISO folder. If you followed steps from the tutorial, your folder will be `/zroot/vm/.iso/`. Also check that you didn't forget to copy the templates into your `/zroot/vm/.templates` folder.

### Create new VM
Let's create a new VM. That can be done easily by running:

```
# vm create -t alpine <vm_name>
```

Replace `<vm_name>` with your instance name. Before you start the installation, take a moment and configure your instance with `vm config <vm_name>`. **You need to replace all occurrences of `vanilla` with `virt`, if you have downloaded Virtual image**. Here's my configuration file at `/zroot/vm/alpine/alpine.conf`:

```
loader="grub"
cpu=2
memory=2G
network0_type="virtio-net"
network0_switch="public"
disk0_type="virtio-blk"
disk0_name="disk0.img"
grub_install0="linux /boot/vmlinuz-virt initrd=/boot/initramfs-virt alpine_dev=cdrom:iso9660 modules=loop,squashfs,sd-mod,usb-storage,sr-mod"
grub_install1="initrd /boot/initramfs-virt"
grub_run0="linux /boot/vmlinuz-virt root=/dev/vda3 modules=ext4"
grub_run1="initrd /boot/initramfs-virt"
uuid="525267a7-1ee9-48d5-9dcf-a03f9b63c3aa"
network0_mac="58:9c:fc:04:c2:99"
```

I call my instance `alpine`. As you can see, I use 2 CPU cores rather than one. I also changed memory to 2G, just in case. I left the rest unchanged. You can also see that I'm using `public` switch as we have created before.

*Note: It should be common sense, but just in case: please do not blindly copy&paste configuration file. Always take your time and educate yourself and use the example only for your inspiration. While there's (usually) nothing wrong with using default configuration, it's a bad habit to blindly use configuration files / commands from the internet.*

### Alpine Linux Installation
Almost there! It's time to spin the installation process and follow the instructions. Start by kicking off the installation process:

```
# vm install <vm_name> <iso_name>
```

Replace `<vm_name>` with name of your VM and `<iso_file>` with the filename of the ISO, without path. In my case it was `alpine-virt-3.15.0-x86_64.iso`. The VM is started in installation mode. Connect to the console:

```
# vm console <vm_name>
```

If you use `tmux` as recommended, you will be welcomed to familiar environment. Once the boot process is done, you will be asked to login. Use `root` without password and then start the installation process:

```
# setup-alpine
```

Once you get into the network configuration, things will become interesting. As you may have noticed, we have no DHCP configured for our NAT. I think in general it's better to use static IP configuration for servers anyway. I did not take a screenshot of the installation process, but I will tell you what values I put there during the configuration. Do not use DHCP as mentioned before and go for static configuration:

```
address: 10.0.1.2
netmask: 255.255.255.0
gateway: 10.0.1.1
```

I use the first available IP (`10.0.1.2`) after gateway (`10.0.1.1`). You will be also asked about host name, feel free to pick whatever you like. When you get asked about DNS, you may use the same settings you have on your main system. If you don't know, you can go with `1.1.1.1`, unless you have a different preference.

When asked about disk, I use `sys` option to keep things simple. I do not use SSH, because I can easily connect via console. I left NTP daemon option on default although I would never use that on BSD. On Linux, my standards (and expectations) are pretty low ;-)

## Docker
### Installation
Okay, here we go. Finally some Docker stuff! Before we can install Docker on our fresh Linux installation, we have to enable Community repository. Head to `/etc/apk/repositories` with your `vi` and uncomment the second line. At the time of writing this article, the content was:

```
#/media/cdrom/apks
http://uk.alpinelinux.org/alpine/v3.15/main
http://uk.alpinelinux.org/alpine/v3.15/community
#http://uk.alpinelinux.org/alpine/edge/main
#http://uk.alpinelinux.org/alpine/edge/community
#http://uk.alpinelinux.org/alpine/edge/testing
```

Notice that since I'm based in the UK, I'm using local mirror. Please use the mirror closest to your location. Now it's time to update APK, install Docker and enable service at boot.

```
# apk update
# apk add docker
# rc-update add docker boot
```

You have now Docker daemon running locally in your Linux, congratulations! The only problem is that it's not available from FreeBSD. Let's fix that. Create new file called `/etc/docker/daemon.json` and put the following configuration there:

```
{"hosts": ["tcp://0.0.0.0:2375", "unix:///var/run/docker.sock"]}
```

Now it's time to start the daemon and verify that all is working well:
```
# service docker start
# docker ps
CONTAINER ID   IMAGE     COMMAND   CREATED   STATUS    PORTS     NAMES
#
```

We do not have any containers yet, but the daemon is up and running. If you chose to use `tmux`, exit the console by `Ctrl-B d`.

### Local Use
From your local terminal in FreeBSD, you may now run Docker commands like this:

```
$ DOCKER_HOST=10.0.1.2 docker ps
CONTAINER ID   IMAGE     COMMAND   CREATED   STATUS    PORTS     NAMES
$ DOCKER_HOST=10.0.1.2 docker run hello-world

Hello from Docker!
This message shows that your installation appears to be working correctly.

To generate this message, Docker took the following steps:
 1. The Docker client contacted the Docker daemon.
 2. The Docker daemon pulled the "hello-world" image from the Docker Hub.
    (amd64)
 3. The Docker daemon created a new container from that image which runs the
    executable that produces the output you are currently reading.
 4. The Docker daemon streamed that output to the Docker client, which sent it
    to your terminal.

To try something more ambitious, you can run an Ubuntu container with:
 $ docker run -it ubuntu bash

Share images, automate workflows, and more with a free Docker ID:
 https://hub.docker.com/

For more examples and ideas, visit:
 https://docs.docker.com/get-started/

```

### Various Improvements
I don't like to remember various IP addresses, so I prefer to edit my `/etc/hosts` file and add the following line there:

```
10.0.1.2        alpine.beastie alpine
```

That will allow me to reference the host only as `alpine.beastie` or just `alpine`.

You may want to also store `DOCKER_HOST` variable in your local shell login file, so you can then just type `docker` command without trailing variable.

## Conclusion
You have now a fully working Docker installation in FreeBSD, congratulations!

It requires a bit of work, some memory and storage overhead. Many developers and sysadmins will most probably prefer Linux when dealing with Docker. For me personally, I'm happy with this solution as it's a convenient way of having Docker on the OS that I love to use.

I must say though that in majority of my use cases, I use FreeBSD Jails instead. It's much faster and easier to work with them, although the use case is of course different than with Docker containers.

## Links
* https://github.com/churchers/vm-bhyve
* https://github.com/churchers/vm-bhyve/wiki/NAT-Configuration
* https://github.com/churchers/vm-bhyve/wiki/Using-tmux
* https://docs.alpinelinux.org/user-handbook/0.1a/Installing/manual.html
* https://wiki.alpinelinux.org/wiki/Docker

