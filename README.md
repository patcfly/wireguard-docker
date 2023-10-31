# wireguard-docker
Wireguard setup in Docker meant for a simple personal VPN.
There are currently 3 flavors:
 - bullseye -  `docker pull docker pull ghcr.io/patcfly/wireguard-docker:main`

Use the flavor (buster or stretch) that corresponds to your host machine if the kernel module install feature is going to be used.

## Overview
This docker image and configuration is my simple version of a wireguard personal VPN, used for the goal of security over insecure (public) networks, not necessarily for Internet anonymity. The debian (stretch and buster) flavors of the image have the ability to install the wireguard kernel module on the host, and the host OS must also use the same version of debian if this feature is going to be used. In addition, the host's /lib/modules directory needs to be mounted on the first run to install the module (see the [Running](#Running) section below). Thanks to [activeeos/wireguard-docker](https://github.com/activeeos/wireguard-docker) and [ghostserverd/wireguard-docker](https://github.com/ghostserverd/wireguard-docker)for the general structure of the docker image - it is the same concept just built on Ubuntu 16.04.


## Configuration
Sample server-side interface configuration to go in `/etc/wireguard` (e.g., `wg0.conf`):
```
[Interface]
Address = 192.168.20.1/24
PrivateKey = <server_private_key>
ListenPort = 5555

[Peer]
PublicKey = <client_public_key>
AllowedIPs = 192.168.20.2
```
Sample client configuration:
```
[Interface]
Address = 192.168.20.2/24
PrivateKey = <client_private_key>
ListenPort = 0 #needed for some clients to accept the config

[Peer]
PublicKey = <server_public_key>
Endpoint = <server_public_ip>:5555
AllowedIPs = 0.0.0.0/0,::/0 #makes sure ALL traffic routed through VPN
PersistentKeepalive = 25
```

## Other Notes
- This Docker image also has a iptables NAT (MASQUERADE) rule already configured to make traffic through the VPN out to the Internet work. This can be disabled by setting the environment variable `IPTABLES_MASQ=0`.
- For some clients (a GL.inet router in my case) you may have trouble with HTTPS (SSL/TLS) due to the MTU on the VPN. Ping and HTTP work fine but HTTPS does not for some sites. This can be fixed with [MSS Clamping](https://www.tldp.org/HOWTO/Adv-Routing-HOWTO/lartc.cookbook.mtu-mss.html). This is simply a checkbox in the OpenWRT Firewall settings interface.
- This image can be used as a "client" as well. If you want to forward all traffic through the VPN (`AllowedIPs = 0.0.0.0/0`), you need to use the `--privileged` flag when running the container

## docker-compose
Sample docker-compose.yml
```yaml
version: "2"
services:
 vpn:
  image: ghcr.io/patcfly/wireguard-docker:main
  volumes:
   - /local/config/wireguard:/etc/wireguard
   - /lib/modules:/lib/modules
  networks:
   - net
  ports:
   - 5555:5555/udp
   - inboundport:otherserviceport
  restart: unless-stopped
  cap_add:
   - NET_ADMIN
   - SYS_MODULE
  sysctls:
   - "net.ipv6.conf.all.disable_ipv6=0"
  # - "net.ipv6.conf.default.forwarding=1"cd 
  # - "net.ipv6.conf.all.forwarding=1"
   - "net.ipv4.ip_forward=1"
  cap_add:
   - NET_ADMIN
   - SYS_MODULE
```
## Run
First run, make sure your container starts and you can 
```sh
docker compose up wireguard 
```

## Build
Since the images are already on Docker Hub, you only need to do this if you want to change something
```sh
git clone https://github.com/cmulk/wireguard-docker.git
cd wireguard-docker

docker build -f Dockerfile -t wireguard:local .
