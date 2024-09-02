---
layout: post
comments: false
title: "WireGuard Port Forwarding"
date:   2024-09-01 17:00:00
---

## 🎯 What's the Goal?

There are many ways to protect your private devices that host public services. This approach here is just one approach I chose to learn two things:

1. WireGuard
2. iptables

Imagine having a service, in this case, a [Ghidra](https://https://ghidra-sre.org/) server, running on your personal computer at home and you'd like to protect your home IP address.

![network-concept](/assets/images/wireguard-port-forwarding/network-concept.png)

## 💡 Different Approaches 

I initially thought using a Cloudflare tunnel would work since I can map a subdomain to a specific port on the private device running the cloudflared daemon. This way, when someone from the internet uses their Ghidra client to connect to my Ghidra server, they would use my URL, which Cloudflare's DNS would redirect the traffic to a Cloudflare public IP address. I erroneously presumed Cloudflare would take an incoming connection on ports 13100-13102 (Ghidra server's ports) and tunnel them to my cloudflared daemon to ports 13100-13102 on my laptop. That's not how Cloudflare works at all. You can only use web ports like 80, 443, or 8080 to tunnel through Cloudflare.

Additionally, one could forego running the Ghidra server on a private home computer and instead run it on an Amazon Web Service or Oracle Cloud instance. This would alleviate all concerns about exposing one's home IP or opening oneself up to vulnerabilities from misconfiguration or what you have. The problem with this approach is...nothing! It is perfectly valid but less of a learning experience for me.

There are other reasonable ways to accomplish this redirection. Of course, there are also many ways to do this unreasonably. We could create a digital Rube Goldberg machine with arbitrary forwarding layers, virtualization, redirection, obfuscation, etc., but that's not a today-problem. 

## 🐉 Port Forwarding with WireGuard

The general construct is the same as before. We want any Ghidra client on the internet to connect to port 13100 on our public server. That public server forwards the traffic to its WireGuard interface and through the WireGuard tunnel to the private computer running the Ghidra server.

The public server I'm running WireGuard on is the free-tier Oracle Cloud instance. The specific iptables may need to differ for other cloud providers, but this is my configuration.

```conf
# Wireguard configuration for public server
[Interface]
Address = 10.10.10.1/24
ListenPort = 51820
PrivateKey = (hidden)

# Allow packet forwarding
PreUp = sysctl -w net.ipv4.ip_forward=1

# Forward traffic from the public IP on port 13100 to the WireGuard peer's IP
PreUp = iptables -t nat -A PREROUTING -i ens3 -p tcp --dport 13100:13102 -j DNAT --to-destination 10.10.10.2:13100-13102
# Allow forwarding from ens3 to wg0
PreUp = iptables -A FORWARD -i ens3 -o wg0 -p tcp --dport 13100:13102 -d 10.10.10.2 -j ACCEPT
# Allow forwarding from wg0 to ens3
PreUp = iptables -A FORWARD -i wg0 -o ens3 -j ACCEPT
# Masquerade the outgoing traffic to ens3 (public interface)
PreUp = iptables -t nat -A POSTROUTING -o ens3 -j MASQUERADE
# Ensure WireGuard traffic is allowed
PreUp = iptables -A INPUT -i ens3 -p udp --dport 51820 -j ACCEPT
PreUp = iptables -A INPUT -i wg0 -j ACCEPT

# Remove rules from iptables when WireGuard stops
PostDown = iptables -t nat -D PREROUTING -i ens3 -p tcp --dport 13100:13102 -j DNAT --to-destination 10.10.10.2:13100-13102
PostDown = iptables -D FORWARD -i ens3 -o wg0 -p tcp --dport 13100:13102 -d 10.10.10.2 -j ACCEPT
PostDown = iptables -D FORWARD -i wg0 -o ens3 -j ACCEPT
PostDown = iptables -t nat -D POSTROUTING -o ens3 -j MASQUERADE
PostDown = iptables -D INPUT -i ens3 -p udp --dport 51820 -j ACCEPT
PostDown = iptables -D INPUT -i wg0 -j ACCEPT

# Remote settings for the private server
[Peer]
PublicKey = BKKk8U2+omjlt80FRdv2jyapbQwFaDdL9cUzbdOZGHo=
AllowedIPs = 10.10.10.2/32
```

Below is the WireGuard configuration to use on the private server running the Ghidra server service

```conf
# Local settings for the private server
[Interface]
PrivateKey = (hidden)
Address = 10.10.10.2/24

# Enable packet forwarding
PreUp = sysctl -w net.ipv4.ip_forward=1

# Remote settings for public server
[Peer]
PublicKey = 03seb420n22XIufEKyCb/8/9GLJtQreNFznu6/Fj5VY=
Endpoint = 64.x.x.x:51820	# This is the public IP address of the public server
AllowedIps = 10.10.10.1/32
PersistentKeepalive = 25
```

The image below adds some logical context to the configuration shown above. Some random computer on the internet connects to our public server on ports 13100-13102. The public server routes those three ports from its `ens3` interface to its `wg0` WireGuard interface, where our private server running the Ghidra server listens.

![logical-network-concept](/assets/images/wireguard-port-forwarding/logical-network-concept.png)

## 🛠️ Configuring the Ghidra Server

The Ghidra server configuration was pretty straightforward. The steps are generally:

1. Download and unpack [Ghidra](https://github.com/NationalSecurityAgency/ghidra/releases)
2. Download and install a Java runtime and development kit (JDK). E.g., Amazon's long-term support for [JDK 21](https://docs.aws.amazon.com/corretto/latest/corretto-21-ug/downloads-list.html)
3. Install the Ghidra server using
   ```bash
   $ sudo [ghidra_dir]/server/svrInstall
   ```
4. Edit ghidra's `[ghidra_dir]/server/server.conf` to fit your usecase
5. Start the ghidra server
   ```bash
   $ sudo [ghidra_dir]/server/ghidraSvr start
   ```
_Note: This isn't an exhaustive list, and there are potentially other steps depending on your specific requirements_

### Ghidra Server Configuration

There are only two specific settings that I changed:
```conf
ghidra.repositories.dir=/srv/ghidra-repos
wrapper.app.parameter.1=-ip 64.#.#.#
wrapper.app.parameter.2=-a0
wrapper.app.parameter.3=-u
wrapper.app.parameter.4=${ghidra.repositories.dir}
```

These settings are pretty straightforward. I'll stress that the `wrapper.app.parameter.1=-ip 64.#.#.#` is a required parameter in this configuration. You have to tell the Ghidra server what public IP or FQDN users will use to connect to the server.

## Conclusion

That's it. Reasonably straightforward setup in the grand scheme of things. This was my first time playing around with WireGuard, and it's incredible how simple and intuitive it is to get set up and create a peer network. There are much more advanced ways to use WireGuard, but I still need to get there. I'm still not comfortable enough with `iptables` where I could teach someone or rattle off a bunch of rules to make some arbitrary scenario work, but generally speaking, the routing and rules make sense.

I wrote this guide as much for my future self as for anyone else. I hope it can be useful to someone and provide a valuable reference if ever needed. 😊