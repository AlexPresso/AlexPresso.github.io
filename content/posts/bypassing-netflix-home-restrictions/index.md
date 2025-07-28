---
title: "Bypassing Netflix’s Home Restrictions"
date: 2025-07-21T22:30:49+02:00
draft: false
---

In 2023, Netflix started to crack down on password sharing.
One way they did this was by introducing the concept of **Netflix Households**.
Now, while I get that running a massive streaming platform like Netflix isn’t cheap, I completely disagree with how they've chosen to enforce this.

Families today don’t always live in the same household. 
Kids move away for college. Parents may live in different cities. Partners travel. That’s exactly the situation I found myself in, and here is how I fixed it.

## Netflix Household

Netflix defines a household using different techniques / checks:
- Device Fingerprinting : To identify a device even when its IP changes, as it could be connected to a different network.
- IP + IPQS (IPQualityScore) : Checking if you're actually connected to the household network, not from an IP range that could be within a proxy/VPN/Datacenter and that your IP is not flagged as malicious/location spoofing by IPQS.
- Location + Activity tracking : To check that device location is (approximately) the same as the household IP location or if your location is changing very fast across the globe.

Device fingerprinting is not really something that need to be focused on, but correctly spoofing a location in a way that is "nearly" undetectable needs a bit more than just a VPN setup.

## Network Architecture

My home network stack is entirely deployed and orchestrated in a [K3S](https://k3s.io/) (Light Kubernetes) cluster on a Raspberry PI :
- [PiHole](https://pi-hole.net/) with a [Cloudflared](https://pi-hole.net/) side-car : For DHCP, DNS Sink (Domain blacklist / Ad-blocker), DoH (DNS-over-HTTPs).
- [Wireguard](https://www.wireguard.com/) : VPN server configured in a split-tunnelling mode, to use PiHole DNS Sink from mobile devices + access my home NAS.

*If you're also using Kubernetes, and willing to deploy the same stack, you can find my helm charts [here](https://github.com/AlexPresso/helm.alexpresso.me/tree/main/charts) (explainations on adding the helm repo are in the Readme)*.

{{< figure src="network-architecture.png" alt="My home network architecture" title="Network Architecture" >}}

## The Solution

The first thing to do to spoof our location is to use a VPN.
The VPN server needs to be hosted in a home-network, that is defined as the Netflix Household IP.
From here, most other posts are suggesting to:
- "Just tunnel all client traffic through your VPN" : it works, but it's not something I like to do, I prefer split-tunneling (only tunnel necessary traffic through the VPN)
- "Tunnel only Netflix IP ranges" : Not safe. It only works for a limited time until one of Netflix backends (which are hosted on AWS) change IP outside of the Amazon range you've configured.
And you have to do it for all devices / enroll devices on an MDM (Mobile Device Management).

The solution I implemented is relying on **DNS Spoofing** and a **Forward Proxy** and has the following benefits :
- It won't be affected by target IP changes
- You can keep split-tunelling
- Minimal maintenance and totally transparent for client devices
- Can work with any other website than Netflix, without changing VPN config, just by doing **DNS Spoofing** for these domains.

### What is DNS Spoofing

DNS Spoofing is a technique used by hackers and companies which consist in replying a different IP than the real domain one to a DNS request, to intercept traffic.
In example, when requesting `netflix.com` IP address by doing a DNS request using a public DNS server (`1.1.1.1` - CloudFlare), you'll get the real Netflix IPs and your client will use it for next requests:
```bash
$ dig netflix.com @1.1.1.1 +short
54.73.148.110
54.155.246.232
18.200.8.190
```
Now, when you have your own DNS server, you're in control and can reply anything you want to this `netflix.com` request:
```bash
$ dig netflix.com +short
192.168.1.108
192.168.1.108
```
That's what I'm doing to route all netflix requests through a transparent **Forward Proxy** hosted on IP `192.168.1.108`.

### What is a Forward Proxy

A **Forward Proxy** is a network component that accepts connections on a private network and forwards them to the public internet. 
It's the opposite of a **Reverse Proxy** which accepts connections originating from public internet and routes them to resources on a private network.

### How Does It Work

Everything here is possible because of only one thing: the SNI (**Server Name Indication**).
<br>When a client makes an HTTPS request, it initiates a [TLS Handshake](https://www.cloudflare.com/learning/ssl/what-happens-in-a-tls-handshake/).
The first packet (also known as **Client Hello**) is not encrypted and always contains an SNI.
This SNI is necessary for a Transparent Forward Proxy (RAW TCP Passthrough) to know where to forward the packets to. 
Without it, it would require the Forward Proxy to play a new request on its own (**TLS Termination Point**), but it 
involves trusting certificates on client devices and won't work because of Netflix [Certificate Pinning](https://developer.android.com/privacy-and-security/security-config). 

Having a plaintext SNI is a serious threat to privacy as anybody monitoring the network can know the domain names of websites you're visiting.
That's why there is a new emerging standard called ECH ([Encrypted Client Hello](https://datatracker.ietf.org/doc/html/draft-ietf-tls-esni-25)), which I'll talk about later.

Here is the full workflow involved when using the transparent Forward Proxy with DNS Spoofing (the VPN Server was ignored to keep clarity):

{{< figure src="full-workflow.png" alt="Full Workflow" title="Full Workflow" >}}
1. The device make a DNS request to resolve a domain IP (i.e. `netflix.com`)
2. PiHole DNS server replies with the Forward Proxy IP
3. The device initiates the connection and sends a **Client Hello** packet to the Forward Proxy
4. The Forward Proxy reads the Client Hello's SNI and makes a new DNS request to resolve the domain real IP
5. The real (non spoofed) DNS Server replies with real IP
6. The Forward Proxy passthrough raw TCP packets to target host

#### Ports Allocation

Before going further, our Forward Proxy will need to bind HTTP/s ports (`80`, `443`). 
If you're deploying everything on the same host, you might be in a situation where those ports are already in use by another service. 
Specially in a Kubernetes environment, for the Ingress Controller.

You can have multiple services/processes use the same host/port, as long as they do not share the same IP.
For this purpose I added two IPs:
```
$ ip addr add 192.168.1.108/24 dev eth0
$ ip -6 addr add fd12:caf3:babe::1/64 dev eth0
```
Those IP addresses won't be persisted across reboot, so you need to make it persistent. 
Depending on your distribution, it should be done either in `ifupdown`, `netplan`, `NetworkManager` or `systemd-networkd`. 
<br>On a Raspberry Pi, it's in `/etc/rc.local`.
<details>
    <summary>/etc/rc.local</summary>

```Bash
#!/bin/sh -e
#
# rc.local
#
# This script is executed at the end of each multiuser runlevel.
# Make sure that the script will "exit 0" on success or any other
# value on error.
#
# In order to enable or disable this script just change the execution
# bits.
#
# By default this script does nothing.

# Print the IP address
_IP=$(hostname -I) || true
if [ "$_IP" ]; then
printf "My IP address is %s\n" "$_IP"
fi

ip addr add 192.168.1.108/24 dev eth0
ip -6 addr add fd12:caf3:babe::1/64 dev eth0

exit 0
```
</details>

#### The Forward Proxy

There are a lot of available proxy technologies out there. 
I've chosen [NGinX](https://nginx.org/en/) because I'm used to work with it but anything is okay.

The most important feature to look after is the ability to control headers sent by the proxy, because normally 
(thanks to [RFC 7239](https://www.rfc-editor.org/rfc/rfc7239.html#section-4) and [RFC 7230](https://www.rfc-editor.org/rfc/rfc7230#section-5.7.1)) all proxies should add a `Via` or `Forwarded` header.
Of course having such headers would make it easily spottable by Netflix and luckily, NGinX adheres to a minimalist header philosophy thus does not add them by default.



## A Note On TLSv1.3 And ECH

