---
title: "Move to OpenBSD"
date: 2024-04-03T19:30:21+02:00
description: "My move from FreeBSD on Vultr to OpenBSD on OpenBSD.amsterdam."
slug: "move-to-openbsd"
---

I decided to move my server from FreeBSD hosted on Vultr to OpenBSD hosted on [OpenBSD.amsterdam](https://openbsd.amsterdam). The main reason was recent change of Vultr's T&C, which says that I'm supposed to give them rights to do with my work whatever they want. I chose OpenBSD.amsterdam hosting, because it's run by passionate people. I also wanted to give OpenBSD a chance, to see how different it is from FreeBSD. 

I already know and use several OpenBSD projects, like OpenSSH, OpenNTPD, OpenSMTPD and of course their excellent packet filter `pf`. I was looking forward to use it's "new" syntax, as it's not (or only partially) available on FreeBSD. FreeBSD jails are not available on OpenBSD. I don't miss them though, as I had alreay enough time play around with them. Plus the complexity of managing them doesn't quite pay off with such a small project as my server.

# Hosting
The process of creating a new VM with OpenBSD.amsterdam was easy and pleasant. They created my VM manually after I contacted them via their contact form. It was a nice change, to have provisioned VM by a human, not a script. I decided to try the VM first, before I made a payment. I installed couple of usual utilities like `zsh` and `vim` and played around with `httpd` first. After few hours I've been convinced enough to make a payment. Their offer not only cheaper than from Vultr, but also they support OpenBSD Foundation as well.

# httpd
My first step was to transfer this blog. I wanted to try OpenBSD's `httpd` instead of usual Apache. I copied `/etc/example/httpd.conf` to `/etc/httpd.conf` in order to have a good starting point. Note that this convention is being repeated for other bundled services too. I started with simple HTTP transport and created a sample HTML file by hand. Everything worked smoothly, so I added TLS support. Take a look at the configuration. See how amazingly simple and short it is, compared to Apache's kilometer long config file:

```
server "boucek.me" {
        listen on * port 80
        block return 301 "https://boucek.me/$REQUEST_URI"
}

server "boucek.me" {
        listen on * tls port 443
        tls {
                certificate "<path_to_fullchain.pem>"
                key "<path_to_private.key>"
        }

        root "/htdocs/boucek.me"

        location "/.well-known/acme-challenge/*" {
                root "/acme"
                request strip 2
        }
}
```

In order to get SSL certificate, I used bundled `acme-client`. I again used the example file `/etc/examples/acme-client.conf` as a starting point. I listened to their suggestion to use staging environment first, before moving to production. It was a good move, because I forgot to correctly update my DNS records first. Even then I had to wait an hour before Let's Encrypt noticed a change in DNS record. Again, see the extremely simple configuration of it:

```
authority letsencrypt {
        api url "https://acme-v02.api.letsencrypt.org/directory"
        account key "<private_key.pem>"
}

authority letsencrypt-staging {
        api url "https://acme-staging-v02.api.letsencrypt.org/directory"
        account key "<staging_private_key.pem>"
}

domain boucek.me {
        alternative names { www.boucek.me }
        domain key "<domain.key>"
        domain certificate "<certificate.crt>"
        domain full chain certificate "<fullchain.pem>"

        # Test with the staging server to avoid aggressive rate-limiting.
        #sign with letsencrypt-staging
        sign with letsencrypt
}
```

The last part was to add `acme-client boucek.me && rcctl reload httpd` to the cron table. That's it, server is up and running. Easy peasy lemon squeezy.

# Wireguard
Second step was to move Wireguard to OpenBSD. Originally I thought I'll just use my old configuration from FreeBSD. But then I found a more elegant way, described by Solene in her article about [Wireguard on OpenBSD](https://dataswamp.org/~solene/2021-10-09-openbsd-wireguard-exit.html). She is using native OpenBSD configuration format to configure the WG interface, which doesn't require to install any external dependency. Here's the content of file `/etc/hostname.wg0`:

```
wgkey <private_key>
wgpeer <public_key> wgaip 10.0.0.2/32

inet 10.0.0.1/24 # used to be /32, see notes below
wgport 51820

up
```

The last part was to set up NAT and allow the traffic in `pf`. Should be easy, right? Well, the actual configuration of firewall was easy. But no traffic was reaching the outside world from Wireguard. It took me a while to realize that I have set up a wrong NAT configuration by accidentally setting /32 mask on NIC for Wireguard. Since I was using it in `pf`, I just couldn't figure out why the heck is such a simple configuration not working. Only after I run `traceroute -s` with the internal IP, I finally realized what I've done. I changed the mask to /24 and everything started working smoothly. Snippet of relevant `pf.conf` parts follows:

```
pass out on egress from (wg0:network) nat-to (vio0:0)          # NAT
pass in log on egress proto udp from any to self port $wg_port # service
pass in on wg0 from (wg0:network)                              # traffic from WG client
```

Notice how I don't specify the network here, I simply take it from the interface itself. Apart from this drama at the end, it was nice and smooth ride. That's all, folks!
