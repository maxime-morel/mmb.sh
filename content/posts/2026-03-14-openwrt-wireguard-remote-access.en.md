---
title: "OpenWrt WireGuard VPN Setup (LAN Access, Full Tunnel, Cloudflare DDNS)"
date: 2026-03-14
lastmod: 2026-03-14
draft: false
translationKey: "openwrt-wireguard-remote-access"

slug: "openwrt-wireguard-vpn-lan-full-tunnel-cloudflare-ddns"
aliases:
  - "/posts/2026-03-14-openwrt-wireguard-remote-access/"
  - "/en/posts/openwrt-wireguard-behind-isp-router/"

description: "Step-by-step OpenWrt WireGuard setup with firewall rules, UDP port forwarding, Cloudflare DDNS, Android QR onboarding, full-tunnel routing, and security checks."
summary: "Practical OpenWrt 24.10.5 WireGuard tutorial (Linksys MR8300): wg0 setup, firewall zones, port 32541 forwarding, Cloudflare DDNS, Android QR setup, troubleshooting, and hardening."

author:
  - "Maxime Morel-Bailly"

tags:
  - "wireguard"
  - "openwrt"
  - "vpn"
  - "network"
  - "android"
  - "cybersecurity"

keywords:
  - "wireguard openwrt"
  - "openwrt vpn"
  - "openwrt wireguard cloudflare ddns"
  - "wireguard android qr code"
  - "openwrt 24.10.5"
  - "linksys mr8300"
  - "huawei hg8145x6"
  - "port forwarding"
  - "dynamic dns"
---

When people think about a VPN, they usually start with privacy on public Wi-Fi or accessing geo-restricted content. My main use case is slightly different: **reach my home network remotely** and, when needed, **send all of my phone or laptop Internet traffic through my house**.

In practice, I want to:

1. access local resources while away from home
2. administer some services without exposing them directly to the Internet
3. use my home connection as the Internet exit point from third-party networks
4. keep the whole design simple, maintainable, and mobile-friendly

For this, **WireGuard** is a strong fit. It is easier to understand and maintain than many older VPN options, performs well, and integrates cleanly with **OpenWrt**.

This article is based on my setup:

- a **Linksys MR8300** with **OpenWrt 24.10.5** (last known supported version)
- the OpenWrt router is placed behind a Huawei **ISP router**
- remote access mainly from **Android**

The goal is to build a VPN setup that provides both:

- access to the **LAN behind OpenWrt**
- as well as **full Internet routing** through the house (`AllowedIPs = 0.0.0.0/0` on the client side)

I do not see WireGuard here as a comfort feature. It is a remote access and administration component, so it deserves a minimum level of network and security discipline.

If you only need the implementation steps, jump to [Step 1](#step-1-create-the-wireguard-interface-in-luci).

This guide is written for the exact query pattern most people have in practice: **OpenWrt WireGuard behind ISP router + Android client + DDNS + full-tunnel Internet**.

## What this setup does

This setup is designed for a home network where:

- the **ISP router stays in place**
- the **OpenWrt router sits behind it**
- one **UDP port** is forwarded from the ISP router to OpenWrt
- clients connect with **WireGuard**
- the VPN can provide both **LAN access** and **full-tunnel Internet**

This matters because many tutorials assume that the OpenWrt router is directly exposed to the Internet. That is not my case here.

## Prerequisites for OpenWrt behind an ISP router

Before starting, I need:

- **OpenWrt 24.10.5** installed on the router
- admin access to both **OpenWrt LuCI** and the **Huawei HG8145X6**
- the ability to create a **UDP port forward** on the ISP router
- a stable WAN IP for the OpenWrt router on the ISP side, ideally via **DHCP reservation**
- a plan for **DynDNS** or another way to reach the home connection if the public IP changes
- WireGuard packages installed on OpenWrt

On my router, I installed:

```bash
opkg update
opkg install kmod-wireguard wireguard-tools luci-proto-wireguard
```

These packages can also be installed directly from the **LuCI** interface through **System > Software**.

In that case, click **Update lists...** first, then filter packages with `wireguard`, and install them from the web interface.

The `luci-proto-wireguard` package is only needed if I want to manage the WireGuard interface and peers directly from **LuCI**. It is therefore not strictly required if the whole setup is done from the command line.

On my router, installing `luci-proto-wireguard` from LuCI also pulled in the other two required packages automatically.

![LuCI software page showing the WireGuard packages after clicking Update lists and filtering for wireguard](/images/openwrt/openwrt-luci-wireguard-packages.png)

*LuCI software page after clicking **Update lists...**, filtering for `wireguard`, and showing `luci-proto-wireguard`, `kmod-wireguard`, and `wireguard-tools`.*

One practical detail: **I had to reboot the router after installing the packages** before the WireGuard protocol showed up in the LuCI interface creation dialog.

If WireGuard does not appear in LuCI after installation, I would first verify:

1. that the packages are really installed
2. that the kernel version matches
3. and then simply that the router has been **rebooted**

## Why use this instead of exposing services directly

If the goal is only to reach one internal service, the common temptation is to publish a port on the ISP router and access that service directly.

The problem is straightforward:

- every exposed service increases the attack surface
- access control becomes scattered
- Internet exposure and administrative access get mixed together
- ISP routers are often too limited to manage this cleanly

With a VPN, the model changes:

- **a single entry point** is exposed
- authentication relies on **cryptographic keys**
- LAN access can remain **private**
- remote administration becomes more coherent

It is not a magic shield, but it is often cleaner than opening several admin ports or exposing internal web interfaces.

## Benefits and drawbacks

Before configuring anything, it helps to be explicit about the trade-offs.

### What this gives me

- remote access to the **home LAN**
- the ability to send **all Internet traffic** through the house
- a relatively lightweight setup on OpenWrt
- strong client support on **Android**, **Linux**, **Windows**, **macOS**, and **iOS**
- a reduced exposure surface with **one UDP port**

### What I still have to accept

- I need to handle **NAT / port forwarding** on the ISP router
- if the public IP changes, I need **DynDNS**
- if the home uplink is weak, tunnel performance will be limited quickly
- full tunnel mode uses more battery on mobile and adds some latency
- a poor configuration can expose too much of the LAN
- troubleshooting is sometimes less intuitive than with one directly exposed service

In short: this is a strong approach for remote access, but it still requires deliberate network design.

## High-level architecture

The logical layout looks like this:

```text
Android / Other client
       |
       |  WireGuard (UDP)
       v
[ Internet ]
       |
       v
[ ISP Router Huawei HG8145X6 ]
       |
       |  forwards one UDP port
       v
[ OpenWrt Router MR8300 ]
       |
       +--> local LAN
       |
       +--> Internet exit through the OpenWrt WAN
```

The key point is that **the OpenWrt router is not directly exposed at the Internet edge**. It sits behind the ISP router, so I need at least:

1. a **UDP port forward** from the ISP router to the OpenWrt WAN IP
2. ideally a **DHCP reservation** or static WAN IP for OpenWrt on the ISP router
3. a stable way to find the public IP, usually **DynDNS**

## Recommended addressing plan

It helps to keep a clean separation between:

- the **LAN** subnet
- the **VPN** subnet
- the WAN addressing

Simple example:

- OpenWrt LAN: `192.168.10.0/24`
- OpenWrt LAN IP: `192.168.10.1`
- WireGuard network: `10.203.0.0/24`
- WireGuard router IP: `10.203.0.1`
- Android client: `10.203.0.2`
- another client (optional): `10.203.0.3`

This helps because:

- routing stays readable
- it avoids reusing a very common home subnet such as `192.168.1.0/24`

I also prefer avoiding VPN ranges that commonly collide with local tooling defaults. For example:

- Docker often uses bridge networks in the `172.17.0.0/16` range and nearby subnets
- libvirt commonly uses `192.168.122.0/24`
- VirtualBox host-only networks are often seen around `192.168.56.0/24`

For that reason, I would rather choose a less common private `/24` in `10.0.0.0/8`, provided it does not already exist elsewhere in my environment.

If your home LAN uses a very common subnet, there is a real risk of overlap when you connect from another network that happens to use the same range.

## Step 1: create the WireGuard interface in LuCI

In **Network > Interfaces**, create a new interface and select **WireGuard VPN** as the protocol.

I would simply call it `wg0`.

![LuCI interface creation dialog with WireGuard VPN selected as the protocol for wg0](/images/openwrt/openwrt-wg0-create.png)

*Creating the `wg0` interface in LuCI and selecting **WireGuard VPN** as the protocol.*

> Note: if **WireGuard VPN** does not appear in the **Protocol** list, reboot the router after installing the three prerequisite packages: `kmod-wireguard`, `wireguard-tools`, and `luci-proto-wireguard`.

After creating the interface, LuCI opens the main WireGuard configuration screen for `wg0`.

To generate the router key pair in LuCI, I used the **Generate new key pair** button directly in the WireGuard interface settings.

![LuCI WireGuard interface showing the Generate new key pair button](/images/openwrt/openwrt-wg0-generate-key-button.png)

*The **Generate new key pair** button in the `wg0` interface configuration.*

![LuCI WireGuard interface after generating a new private/public key pair](/images/openwrt/openwrt-wg0-generated-keypair.png)

*The `wg0` interface after LuCI has generated a new WireGuard key pair.*

At this stage, I set the main interface values:

- **private key: generated by OpenWrt**

If you prefer to generate keys from the command line, I can also do it with `wg`:

```bash
umask 077
wg genkey | tee privatekey | wg pubkey > publickey
```

This creates a `privatekey` for the interface private key and `publickey` for the peer configuration on the other side
If the setup is done entirely on OpenWrt through the shell, this is the simplest alternative to the LuCI button.

- **listen port**: use a non-default high UDP port, for example `32541/UDP`

I avoid the default WireGuard port when exposing the service to the Internet. This is **not** a security boundary by itself, but it does reduce trivial background scanning and noisy reconnaissance on the default port.

- **interface address**: `10.203.0.1/24`

![Screenshot of main wg0 configuration screen with 10.203.0.1/24 and the chosen listen port.](/images/openwrt/openwrt-wg0-port-and-ip.png)

Another step is to disable the default gateway in the advanced settings of the interface. This option is useful only when OpenWrt itself is acting as a VPN client and must change its own default route through the tunnel, which is not the case in this server setup.

![Screenshot of advanced wg0 configuration screen with disable default gateway.](/images/openwrt/openwrt-wg0-disable-default-gateway.png)

Regarding the **Firewall Settings** tab, at this stage I can only attach `wg0` to an existing zone; in Step 2, I will create a dedicated `vpn` zone and show how to assign `wg0` to it properly.

![Screenshot of VPN wg0 configuration screen with default configuration.](/images/openwrt/openwrt-wg0-vpn-zone-default.png)

And you may now save the interface configuration:

![Screenshot of VPN wg0 configuration saving button.](/images/openwrt/openwrt-wg0-save-interface.png)

## Step 2: attach the interface to a firewall zone

This is a central step. A WireGuard interface without a clear firewall policy is not a finished configuration.

The clean approach is usually to create a dedicated zone, for example `vpn`, and explicitly define what it may reach.

Recommended approach:

- if using LuCI, go to `/cgi-bin/luci/admin/network/firewall`
- add a new `vpn` zone (`Add` button at the bottom of the screen)
- attach `wg0` to it
- set `input = ACCEPT`, `output = ACCEPT`, and `intra-zone forward = REJECT`
- keep masquerading disabled on the `vpn` zone (NAT stays on `wan`)
- allow forwarding from `vpn -> lan` and also forwarding from `vpn -> wan` since I want full Internet routing

For this zone popup, the practical values are:

- **Name**: `vpn`
- **Input**: `accept`
- **Output**: `accept`
- **Intra zone forward**: `reject`
- **Masquerading**: unchecked
- **MSS clamping**: unchecked
- **Covered networks**: `wg0`
- **Allow forward to destination zones**: `lan` and `wan`
- **Allow forward from source zones**: leave empty / `unspecified`
- **Advanced tab**: keep `Covered devices` and `Covered subnets` empty, disable `IPv6 Masquerading`, and use `Restrict to address family = IPv4 only` for an IPv4-only network

![Screenshot of the new OpenWrt vpn zone configuration for WireGuard.](/images/openwrt/openwrt-new-vpn-zone-configuration.png)

And you can now save the new zone:

![Screenshot of VPN wg0 configuration saving button.](/images/openwrt/openwrt-vpnzone-save.png)

### Security note

If you do not need complete access to the whole LAN, you should **not allow everything by default**. The cleaner approach is to restrict access through:
- specific firewall rules
- network segmentation / VLANs
- limiting the reachable services

If the only goal is to administer a few devices, full LAN access may not be the right choice.

## Step 3: allow the WireGuard port on OpenWrt

If the `vpn` zone is correctly linked, I still need to ensure that the router accepts incoming **UDP** traffic on the WireGuard listen port.

On OpenWrt, that is usually done through the WAN zone or a dedicated rule.

In **Firewall > Traffic Rules**, create a rule with:

- **Name**: `Allow-WireGuard-Inbound`
- **Protocol**: `UDP`
- **Source zone**: `wan`
- **Destination**: `This device` (not the `vpn` zone)
- **Destination port**: `32541`
- **Action**: `Accept input`
- **Advanced tab**: `Restrict to address family = IPv4 only` for an IPv4-only setup

![OpenWrt traffic rule allowing inbound WireGuard UDP traffic from WAN to this device.](/images/openwrt/opnwrt-fw-allow-wireguard-inbound-main-settings.png)

And in the same way than previous steps, save the configuration before moving forward with the next step.

## Step 4: forward the port on the ISP router

Because the OpenWrt router sits behind the ISP router, I now need to translate that on the ISP side. The idea is simple:
- the ISP router receives the traffic from the Internet
- it forwards `UDP/32541` to the OpenWrt WAN IP

Recommended prerequisites:
- reserve the OpenWrt address in the ISP router DHCP and make sure that the address does not change
- make sure there is no conflicting rule or UPnP behavior interfering with the flow

On my ISP router, I configured port mapping with:

- **Type**: user-defined (or custom equivalent). In my case the port selection was not available with user-defined, I had to select another random type.
- **Enable Port Mapping**: enabled
- **Internal Host**: OpenWrt WAN IP (example: `192.168.3.8`)
- **Protocol**: `UDP`
- **External port number**: `32541` to `32541`
- **Internal port number**: `32541` to `32541`
- **External source IP address**: empty (unless I explicitly want to whitelist a fixed source IP, this is not my case)

![ISP router IPv4 port mapping for WireGuard UDP 32541 to OpenWrt WAN IP.](/images/openwrt/openwrt-isp-router-port-mapping-wireguard.png)

### Should I put the OpenWrt router in the ISP router DMZ?

Usually, **no**. For this use case, a targeted WireGuard port forward is cleaner than placing OpenWrt in a broad DMZ.

A DMZ can be useful for testing, but for a normal setup I would rather keep:

- a fixed OpenWrt WAN IP
- one targeted port forward
- the minimum exposed surface

## Step 5: handle the public IP with DynDNS

If the ISP provides a static public IP, this is optional. But in many cases the public IP can change every 24 hours or after an ISP router reboot.
Even with a fixed IP, I still prefer a stable DNS name because it avoids touching every client profile later.

In my case, I use my own domain and host DNS on Cloudflare. I made this choice because DynHost was not available in my registrar/manager context, so I migrated the DNS zone to Cloudflare and configured DDNS updates from OpenWrt. You might still be able to use your registrar's DynDNS directly if it is available.

Target hostname example (I recommend choosing a unique subdomain name):
- `myprivatevpnXYZ.mmb.sh`

Once the domain was migrated to Cloudflare, I followed these steps:

### 5.1 Create the WireGuard DNS record

In Cloudflare DNS:

- create an `A` record for `myprivatevpnXYZ.mmb.sh`
- set it to any placeholder IPv4 initially
- set **Proxy status = DNS only** (important for WireGuard UDP)

### 5.2 Create a restricted Cloudflare API token

Create an API token with least privilege:

- permission: `Zone - DNS - Edit`
- permission: `Zone - Zone - Read`
- zone resource: only `mmb.sh`

Store this token securely. It is the secret OpenWrt will use for DDNS updates.

### 5.3 Install DDNS and CA packages on OpenWrt

```bash
opkg update
opkg install ddns-scripts luci-app-ddns ddns-scripts-cloudflare ca-bundle ca-certificates
```

Note that with the LuCI package, **you will need to reboot OpenWrt**.

### 5.4 Configure DDNS in LuCI

Once rebooted, in **Services > Dynamic DNS**, create a new service (IPv4):

- **Service Name**: for example `CloudflareDynDNS`
- **DDNS Service provider**: `cloudflare.com-v4`

![OpenWrt Dynamic DNS page: first step to add a Cloudflare DDNS service.](/images/openwrt/openwrt-ddns-add-service.png)

Then in the new popup that opens:
- **Enable**: checked
- **Lookup Hostname**: `myprivatevpnXYZ.mmb.sh`
- **Domain**: `myprivatevpnXYZ@mmb.sh` (record@zone format)
- **Username**: `Bearer`
- **Password**: `<Cloudflare API Token>`
- **Use HTTP Secure**: checked
- **Path to CA-Certificate**: `/etc/ssl/certs`

![OpenWrt DDNS service popup - Basic Settings for Cloudflare.](/images/openwrt/openwrt-ddns-add-service-basic-settings.png)

In the same DDNS service popup:

- **Advanced Settings**
- if OpenWrt is directly connected to the Internet with a public WAN, `IP address source = Network` and `Network = wan` is fine
- if OpenWrt is behind an ISP router (private WAN like `192.168.x.x`), such as in my case, then use `IP address source = URL` (or `Web`) to detect the real public IPv4
- for URL/Web detection, use a reliable IPv4 endpoint such as `http://checkip.amazonaws.com` or `https://api.ipify.org`
- keep `DNS-Server` empty unless you have a specific reason to override it
- if `Lookup Hostname` resolves to a private IP in logs (for example `192.168.x.x`), set a public resolver explicitly (for example `1.1.1.1`) in `DNS-Server`
- optional: keep `Log to file` enabled for troubleshooting

![OpenWrt DDNS service popup - Advanced Settings for Cloudflare.](/images/openwrt/openwrt-ddns-add-service-advanced-settings.png)

- **Timer Settings**
- set `Check Interval = 5` and `Check Unit = minutes`
- set `Force Interval = 72` and `Force Unit = hours`
- set `Error Retry Interval = 60` and `Retry Unit = seconds`
- keep `Error Max Retry Counter = 0` (unless you want a different retry policy)

![OpenWrt DDNS service popup - Timer Settings for Cloudflare.](/images/openwrt/openwrt-ddns-add-service-timer-settings.png)

Then save and start the service.

Compatibility note: OpenWrt `ddns-scripts-cloudflare` behavior depends on version.
The setup above (`Username = Bearer` with `record@zone`) is the one that worked in my environment.

If logs show headers `X-Auth-Email` and `X-Auth-Key`, your build expects **legacy auth**:

- `Username = Cloudflare account email`
- `Password = Cloudflare Global API Key`
- `Domain = zone` (example: `mmb.sh`)
- `Lookup Hostname = full record` (example: `myprivatevpnXYZ.mmb.sh`)

If logs show `Authorization: Bearer ...`, keep the token-based setup.

### 5.5 Validate updates

You can see the status after clicking "Reload" and checking the logs in the tab `Log File Viewer`. But you can also try these useful checks:

```bash
nslookup myprivatevpnXYZ.mmb.sh 1.1.1.1
logread -e ddns
```

If the value returned by `nslookup` matches the current public IPv4, DDNS is working.

### DynDNS and CGNAT reminder

DynDNS only solves *name-to-IP updates*. If I am behind **CGNAT**, inbound WireGuard may still fail even with perfect DDNS.
In that case, I need another strategy (public IP option, VPS relay, or outbound tunnel design).

## Step 6: create the peers on OpenWrt

Go back to your `wg0` network interface configuration. In the `Peers` tab, you will be able to add peers that will be allowed to connect to the VPN.

Each client should have:

- its own key pair
- its own VPN address
- its own peer entry on the router

I strongly recommend:

- **one key pair per device**
- **one peer per device**
- **never sharing one profile** across multiple devices

### Android peer

Simple example:

- name: `android-pixel`
- generate a key pair (button in the interface)
- allowed IPs for that peer, for example `10.203.0.2/32`

![OpenWrt WireGuard peer settings example for an Android client.](/images/openwrt/openwrt-wg0-android-peer-settings.png)

### Security note

On the router side, `Allowed IPs` for a peer do not only define destinations. They also define **which VPN address belongs to which client**.

If this is configured too broadly or inconsistently, routing becomes harder to reason about and mistakes become easier.

## Step 7: configure the Android client

On Android, the official WireGuard app is usually the simplest option:
- [WireGuard install page (official)](https://www.wireguard.com/install/)

From the OpenWrt peer page:
1. fill and save the peer first
2. click **Generate configuration...**
3. scan the generated QR code from the Android WireGuard app

![OpenWrt generated Android WireGuard configuration with QR code.](/images/openwrt/openwrt-wg0-android-qr-export.png)

The generated profile should include:

- `Address = 10.203.0.2/32`
- `Endpoint = <your-ddns-hostname>:<your-wireguard-port>`
- `AllowedIPs = 0.0.0.0/0, ::/0` for full tunnel
- optional `PersistentKeepalive = 25` for mobile NAT stability

In a full-tunnel case, `AllowedIPs = 0.0.0.0/0, ::/0` sends Internet traffic into the tunnel.

On Android, enable the tunnel. Then, while disconnected from your home Wi-Fi (for example on mobile data), verify that your public IP matches your home public IP.

### My Android caution point

If I keep a permanent full tunnel enabled, I want to watch:

- battery life
- behavior when switching between Wi-Fi and 4G / 5G
- apps that behave differently when the apparent public IP changes

In some cases, having **two profiles** makes sense:

1. one “LAN only” profile
2. one “full tunnel” profile

## Step 8: configure other devices (same principle)

For other clients (Linux, Windows, macOS, iOS), the process is almost the same:

1. create a dedicated peer on OpenWrt (one key pair and one tunnel IP per device)
2. export or manually copy the client configuration
3. set the same endpoint hostname/port and matching keys
4. choose `AllowedIPs` based on your goal (LAN-only vs full tunnel)
5. test handshake and traffic before considering the setup finished

The UI differs by platform, but the WireGuard logic stays identical.

## Tests to run before calling the setup done

I would not stop at “the tunnel connects”.

At minimum, I would verify:

1. that the WireGuard handshake appears on the router
2. that the client can reach `10.203.0.1`
3. that it can reach the LAN, for example `192.168.10.1` and then an internal host
4. that Internet traffic really exits through the house
5. that DNS resolution is coherent
6. that reconnection works after a mobile or Wi-Fi network change

## Common issues and troubleshooting

There are often no explicit errors when the connection fails. You may only notice that Internet access is not working on the client.
If the tunnel does not establish:

1. verify endpoint hostname and UDP port in the generated profile
2. verify OpenWrt listen port and firewall rule use the same port
3. verify the peer public key on OpenWrt matches the phone key pair
4. save the peer before generating the QR code
5. if still failing, regenerate the peer and export/scan a fresh QR code

In my own troubleshooting, points 4 and 5 were exactly what fixed the issue.

### No handshake at all

If the peer never handshakes, I would verify:

- the public endpoint and UDP port
- the port forward on the ISP router
- the OpenWrt firewall rule allowing inbound UDP traffic
- whether the home connection is behind **CGNAT**
- whether the client is using the correct public key for the router

### Handshake works, but I cannot reach the LAN

If the tunnel comes up but the LAN stays unreachable, I would check:

- the `vpn -> lan` forwarding rule
- the peer `Allowed IPs` on OpenWrt
- the LAN subnet used in the client config
- whether the LAN overlaps with the network I am currently connected from

### LAN works, but Internet does not go through the tunnel

If internal access works but public traffic still exits locally, I would check:

- whether the client uses `AllowedIPs = 0.0.0.0/0`
- whether `vpn -> wan` forwarding is allowed
- whether masquerading/NAT behavior is correct on the router
- whether DNS is still pointing to an external resolver instead of the home side

### DNS works poorly or leaks outside the tunnel

If websites open inconsistently or DNS does not follow the tunnel, I would verify:

- the `DNS` value in the client config
- whether the chosen DNS server is reachable through the tunnel
- whether Android is overriding DNS behavior

### The tunnel works from Wi-Fi but not from mobile data

That usually points to NAT behavior, filtering, or a stale endpoint path. In that case I would test:

- `PersistentKeepalive = 25`
- another external network
- whether the ISP router or mobile carrier filters unusual UDP flows

## IPv6 note

The example client profiles include `::/0` in `AllowedIPs`, which is appropriate only if I actually want IPv6 traffic to follow the tunnel as well.

In practice, I should decide this explicitly:

- if my setup supports IPv6 end to end, I can keep `::/0`
- if my home network does not properly route IPv6 through WireGuard, I should remove `::/0` rather than pretend IPv6 is covered

## Full-tunnel routing: useful, but not free

Sending all Internet traffic through the house is useful in several cases:

- access to resources that only accept the home public IP
- continuity between internal and external services
- safer usage on untrusted networks
- more consistent browsing when traveling

But there are limits:

- everything now depends on the home Internet connection
- the home uplink becomes a bottleneck
- a power outage at home also kills that Internet exit point
- some use cases gain little from being tunneled all the time

For a smartphone, the best trade-off is not always “full tunnel all the time”.

## Security practices I would recommend

If I wanted to harden this setup, I would prioritize:

1. one key pair per device and immediate revocation if a device is lost
2. a VPN subnet distinct from the LAN
3. firewall rules limited to the real need
4. disabling UPnP if it is not required
5. regular OpenWrt updates
6. exported backups of the router configuration
7. careful thought about administrative interfaces exposure
8. verifying that IPv6 is either correctly routed or intentionally excluded

## Reference documentation

For exact implementation details and ongoing updates, I would also keep these references nearby:

- [WireGuard Quick Start](https://www.wireguard.com/quickstart/)
- [OpenWrt documentation](https://openwrt.org/docs/start)
- [OpenWrt WireGuard package page](https://openwrt.org/packages/pkgdata/luci-proto-wireguard)
- [OpenWrt Dynamic DNS client](https://openwrt.org/docs/guide-user/services/ddns/client)
- [Cloudflare API tokens](https://developers.cloudflare.com/fundamentals/api/get-started/create-token/)

## Why not Tailscale or ZeroTier?

This setup is specifically about running a **self-managed WireGuard server on OpenWrt**. Tools such as Tailscale or ZeroTier or equivalent can be easier to deploy because they simplify NAT traversal and device enrollment.

I still prefer this OpenWrt + WireGuard approach when I want:

- direct control over the router-side configuration
- no dependency on an overlay control plane
- a predictable path for **full-tunnel routing through the house**
- a setup that stays close to standard WireGuard behavior

If the priority were only “make remote access work quickly”, Tailscale would be worth considering. If the priority is a router-centric setup I understand end to end, WireGuard on OpenWrt is a better fit.

## Conclusion

Running WireGuard on OpenWrt behind an ISP router is entirely realistic, including in a typical home setup. The hard part is usually not the protocol installation itself, but the surrounding design:

- clean addressing
- coherent firewalling
- well-understood port forwarding
- realistic handling of the public IP
- a clear distinction between LAN access and full Internet routing

For my use case, this is cleaner than exposing internal services one by one, and more flexible than relying on a handful of published ports.

Before calling this setup finished, I would validate four things from outside the home network: the real handshake path, LAN reachability, full-tunnel Internet routing, and whether the VPN exposes more administrative surface than intended.

My next step is to lock down the final hardening choices so this draft becomes a publishable field report rather than just a working template.
