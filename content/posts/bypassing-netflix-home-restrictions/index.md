---
title: "Bypassing Netflix’s Home Restrictions"
date: 2025-07-21T22:30:49+02:00
draft: true
future: true
---

> "Love is sharing a password." — [Netflix, on Twitter - 2017-03-10](https://x.com/netflix/status/840276073040371712)

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

The first thing to do to spoof a location is to use a VPN.
The VPN server needs to be hosted in a home-network, that is defined as the Netflix Household IP.
From here, most other posts suggests to:
- "Just tunnel all client traffic through your VPN" : it works, but it's not something I like to do, I prefer split-tunneling (only tunnel necessary traffic through the VPN)
- "Tunnel only Netflix IP ranges" : Not safe. It only works for a limited time until one of Netflix backends (which are hosted on AWS) change IP outside the Amazon range you've configured.
And you have to do it for all devices / enroll devices on an MDM (Mobile Device Management).

The solution I implemented is relying on **DNS Spoofing** with a **Forward Proxy** and has the following benefits :
- It won't be affected by target IP changes (Proxy resolves domains from home-network)
- You can keep split-tunelling
- Minimal maintenance and totally transparent for client devices
- Can work with any other website than Netflix, without changing VPN config, just by doing **DNS Spoofing** for these domains.

### What is DNS Spoofing

DNS Spoofing is a technique used by hackers and companies which consist in replying a different IP than the real domain one to a DNS request, to intercept/monitor traffic.
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
involves trusting certificates on client devices and won't work because of Netflix App [Certificate Pinning](https://developer.android.com/privacy-and-security/security-config). 

Having a plaintext SNI is a serious threat to privacy, that's why there is a new emerging standard called ECH ([Encrypted Client Hello](https://datatracker.ietf.org/doc/html/draft-ietf-tls-esni-25)), which I'll talk about later.

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

#### Forward Proxy Setup

There are a lot of available proxy technologies out there. 
I've chosen [NGinX](https://nginx.org/en/) because I'm used to work with it but anything is okay.

The most important feature to look after is the ability to control headers sent by the proxy, because normally 
(thanks to [RFC 7239](https://www.rfc-editor.org/rfc/rfc7239.html#section-4) and [RFC 7230](https://www.rfc-editor.org/rfc/rfc7230#section-5.7.1)) all proxies should add a `Via` or `Forwarded` header, which are very often hardcoded.
Of course having such headers would make it easily spottable by Netflix and luckily, NGinX adheres to a minimalist header philosophy thus does not add them by default.

Again, I deployed everything in a Kubernetes cluster, you can find the helm chart for this `nginx-sni-proxy` [here](https://github.com/AlexPresso/helm.alexpresso.me/tree/main/charts/nginx-sni-proxy).
<br>If you're not using Kubernetes, you can run a docker NGinX container with the following configuration:
<details>
    <summary>/etc/nginx/nginx.conf</summary>

```Bash
    worker_processes auto;

    events {
      worker_connections 1024;
    }

    stream {
      # Enable SNI extraction from TLS ClientHello
      ssl_preread on;

      resolver {{ join " " .Values.nginx.config.dns.servers }} {{ if not .Values.nginx.config.dns.allowIPv6 }} ipv6=off{{ end }} valid={{ .Values.nginx.config.dns.cache }};

      log_format basic '[$time_local] $remote_addr -> $ssl_preread_server_name:$server_port '
                       'proto=$protocol bytes_sent=$bytes_sent bytes_received=$bytes_received duration=${session_time}s';

      server {
        listen 443;
        proxy_pass $ssl_preread_server_name:443;
        access_log /dev/stdout basic;
      }
    }

    http {
      resolver {{ join " " .Values.nginx.config.dns.servers }} {{ if not .Values.nginx.config.dns.allowIPv6 }} ipv6=off{{ end }} valid={{ .Values.nginx.config.dns.cache }};

      log_format basic '[$time_local] $remote_addr -> $host:$server_port '
                       'proto=$server_protocol bytes_sent=$bytes_sent bytes_received=$request_length duration=${request_time}s';

      server {
        listen 80;
        access_log /dev/stdout basic;

        location / {
          proxy_pass http://$host;
        }
      }
    }
```
</details>

This configuration runs a forward proxy for both `http` and `https` (the `stream` block is what allows RAW TCP Passthrough). 

#### Configuring DNS Spoofing

Now that the forward proxy is deployed, Netflix domains entries need to be added to the DNS server, pointing to your Forward Proxy IPs (in my case `192.168.1.108` and `fd12:caf3:babe::1`).

Of course, manually adding a record for every single domain and subdomain would be a long and unsafe task. For this purpose, wildcard domain entries (i.e. `*.netflix.com`) are perfect, but PiHole
does not natively support it, which is totally understandable, that use case is only for DNS Spoofing, which in general you want to prevent.

Fortunately, PiHole relies on [dnsmasq](https://thekelleys.org.uk/dnsmasq/doc.html) which does handle "wildcard" domain entries:
<details>
    <summary>/etc/dnsmasq.d/1-additional.conf</summary>

```Bash
address=/netflix.net/192.168.1.108
address=/netflixstudios.com/192.168.1.108
address=/netflix.com/192.168.1.108
address=/fast.com/192.168.1.108
address=/netflix.ca/192.168.1.108
address=/netflix.com/192.168.1.108
address=/netflix.net/192.168.1.108
address=/netflixinvestor.com/192.168.1.108
address=/nflxext.com/192.168.1.108
address=/nflximg.com/192.168.1.108
address=/nflximg.net/192.168.1.108
address=/nflxsearch.net/192.168.1.108
address=/nflxso.net/192.168.1.108
address=/nflxvideo.net/192.168.1.108
address=/netflixdnstest*.net/192.168.1.108
address=/amazonaws.com/192.168.1.108

address=/netflix.net/fd12:caf3:babe::1
address=/netflixstudios.com/fd12:caf3:babe::1
address=/netflix.com/fd12:caf3:babe::1
address=/fast.com/fd12:caf3:babe::1
address=/netflix.ca/fd12:caf3:babe::1
address=/netflix.com/fd12:caf3:babe::1
address=/netflix.net/fd12:caf3:babe::1
address=/netflixinvestor.com/fd12:caf3:babe::1
address=/nflxext.com/fd12:caf3:babe::1
address=/nflximg.com/fd12:caf3:babe::1
address=/nflximg.net/fd12:caf3:babe::1
address=/nflxsearch.net/fd12:caf3:babe::1
address=/nflxso.net/fd12:caf3:babe::1
address=/nflxvideo.net/fd12:caf3:babe::1
address=/netflixdnstest*.net/fd12:caf3:babe::1
address=/amazonaws.com/fd12:caf3:babe::1
```
</details>

Or using the [helm chart](https://github.com/AlexPresso/helm.alexpresso.me/tree/main/charts/pihole):

<details>
    <summary>values.yaml</summary>

```yaml
  # ...
  additionalDnsmasq:
    # IPv4
    - "address=/netflix.net/192.168.1.108"
    - "address=/netflixstudios.com/192.168.1.108"
    - "address=/netflix.com/192.168.1.108"
    - "address=/fast.com/192.168.1.108"
    - "address=/netflix.ca/192.168.1.108"
    - "address=/netflix.com/192.168.1.108"
    - "address=/netflix.net/192.168.1.108"
    - "address=/netflixinvestor.com/192.168.1.108"
    - "address=/nflxext.com/192.168.1.108"
    - "address=/nflximg.com/192.168.1.108"
    - "address=/nflximg.net/192.168.1.108"
    - "address=/nflxsearch.net/192.168.1.108"
    - "address=/nflxso.net/192.168.1.108"
    - "address=/nflxvideo.net/192.168.1.108"
    - "address=/netflixdnstest*.net/192.168.1.108"
    - "address=/amazonaws.com/192.168.1.108"
    # IPv6
    - "address=/netflix.net/fd12:caf3:babe::1"
    - "address=/netflixstudios.com/fd12:caf3:babe::1"
    - "address=/netflix.com/fd12:caf3:babe::1"
    - "address=/fast.com/fd12:caf3:babe::1"
    - "address=/netflix.ca/fd12:caf3:babe::1"
    - "address=/netflix.com/fd12:caf3:babe::1"
    - "address=/netflix.net/fd12:caf3:babe::1"
    - "address=/netflixinvestor.com/fd12:caf3:babe::1"
    - "address=/nflxext.com/fd12:caf3:babe::1"
    - "address=/nflximg.com/fd12:caf3:babe::1"
    - "address=/nflximg.net/fd12:caf3:babe::1"
    - "address=/nflxsearch.net/fd12:caf3:babe::1"
    - "address=/nflxso.net/fd12:caf3:babe::1"
    - "address=/nflxvideo.net/fd12:caf3:babe::1"
    - "address=/netflixdnstest*.net/fd12:caf3:babe::1"
    - "address=/amazonaws.com/fd12:caf3:babe::1"
  # ...
```
</details>

At this point, all your local devices should already be acessing Netflix through your Forward Proxy:
```
10.42.0.1 -> occ-0-XXXX-XXXX.1.nflxso.net:443 proto=TCP bytes_sent=24438 bytes_received=991 duration=2.510s
10.42.0.1 -> occ-0-XXXX-XXXX.1.nflxso.net:443 proto=TCP bytes_sent=1141422 bytes_received=1202 duration=3.895s
10.42.0.1 -> occ-0-XXXX-XXXX.1.nflxso.net:443 proto=TCP bytes_sent=572895 bytes_received=1635 duration=5.021s
10.42.0.1 -> occ-0-XXXX-XXXX.1.nflxso.net:443 proto=TCP bytes_sent=67514 bytes_received=991 duration=2.431s
10.42.0.1 -> occ-0-XXXX-XXXX.1.nflxso.net:443 proto=TCP bytes_sent=118640 bytes_received=991 duration=3.550s
10.42.0.1 -> ipv4-XXXX-XXXXXX-ix.1.oca.nflxvideo.net:443 proto=TCP bytes_sent=91456 bytes_received=1444 duration=0.982s
10.42.0.1 -> ipv4-XXXX-XXXXXX-X-isp.1.oca.nflxvideo.net:443 proto=TCP bytes_sent=342429 bytes_received=3108 duration=4.524s
10.42.0.1 -> ipv4-XXXX-XXXXXX-X.1.oca.nflxvideo.net:443 proto=TCP bytes_sent=112581 bytes_received=1834 duration=3.360s
10.42.0.1 -> ios.prod.ftl.netflix.com:443 proto=TCP bytes_sent=5996 bytes_received=2087 duration=0.702s
10.42.0.1 -> ipv4-XXXX-XXXXXX-X.1.oca.nflxvideo.net:443 proto=TCP bytes_sent=27082 bytes_received=1214 duration=2.977s
10.42.0.1 -> ipv4-XXXX-XXXXXX-X.1.oca.nflxvideo.net:443 proto=TCP bytes_sent=267028 bytes_received=3680 duration=5.021s
10.42.0.1 -> ipv4-XXXX-XXXXXX-X-isp.1.oca.nflxvideo.net:443 proto=TCP bytes_sent=257206 bytes_received=2483 duration=4.903s
```

#### Configuring Wireguard

For this setup to work from outside a local network, we need to configure wireguard in a way that each VPN client will tunnel at least:
- DNS requests to PiHole
- Packets going to our Forward Proxy

The following is a very basic split-tunneling configuration, please extend it to your own needs ([wireguard official documentation](https://www.wireguard.com/quickstart/)).

In case you want to setup a firewall configuration to prevent/allow routing from/to specific hosts, it's also possible using `PostUp` scripts, you can find an example in the [helm chart values](https://github.com/AlexPresso/helm.alexpresso.me/blob/main/charts/wireguard/values.yaml#L37). 

<details>
    <summary>server wg0.conf</summary>

```yaml
[Interface]
Address = 10.6.0.1/24, fd10:6::1/64
PrivateKey = # Your Server Private Key
ListenPort = 51820
Table = off
DNS = 192.168.1.107 # PiHole DNS Server IP
MTU = 1420

[Peer]
PublicKey = # Your Peer public key
AllowedIPs = 10.6.0.2/32, fd10:6::2/128
PersistentKeepAlive = 25
PresharedKey = # Your Peer PSK
# ...
```
</details>

<details>
    <summary>client wg0.conf</summary>

```yaml
[Interface]
Address = 10.6.0.2/32, fd10:6::2/128
PrivateKey = # Your Client Private Key
DNS = 192.168.1.107  # PiHole DNS Server IP
MTU = 1420

[Peer]
PublicKey = # Your Server Public Key
Endpoint = # Your DynDNS or home static IP :51820
AllowedIPs = 192.168.1.107/32, 192.168.1.108/32, fd12:caf3:babe::1/128 # Tunnel only PiHole + Forward Proxy traffic
PersistentKeepAlive = 25
PresharedKey = # Your Peer PSK
```
</details>



## A Note On TLSv1.3 And ECH

As said before having a plaintext CH (**Client Hello**) is a serious threat to privacy,
as anybody monitoring the network can know the domain names of websites you're visiting by just monitoring the SNI (**Server Name Indication**).

While ECH (**Encrypted Client Hello**) is not yet officially supported, it aims to be. Some countries like [Russia](https://portal.noc.gov.ru/ru/news/2024/11/07/%D1%80%D0%B5%D0%BA%D0%BE%D0%BC%D0%B5%D0%BD%D0%B4%D1%83%D0%B5%D0%BC-%D0%BE%D1%82%D0%BA%D0%B0%D0%B7%D0%B0%D1%82%D1%8C%D1%81%D1%8F-%D0%BE%D1%82-cdn-%D1%81%D0%B5%D1%80%D0%B2%D0%B8%D1%81%D0%B0-cloudflare/) 
already censored its usage, as it would need direct traffic interception (a **TLS Termination Point**) to read SNIs.

Now, you might be thinking :
> "Okay, ECH is a good thing, but does it mean this technique to bypass Netflix won't work anymore ?"

Short answer: Yes, but it's still possible to achieve SNI-like forwarding, without it. 

I'll make a full article about it when I'll be done coding it.
Until then, take care and have fun watching the flix.


