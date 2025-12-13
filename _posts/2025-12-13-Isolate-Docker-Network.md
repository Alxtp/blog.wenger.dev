---
title: "Isolate Docker Container Networks with Firewalld"
date: 2025-12-13 21:00:00 +0200 
categories: [Network]
tags: [docker, firewalld]
---

In order to increase security when running containers that host public services, it makes sense to isolate the virtual Docker networks from the real network where the host resides. When a container is compromised, the attack surface should be as small as possible and access to the local network denied. In order to achieve isolation, it makes sense to use firewalld, as Docker has an integration: [Integration with firewalld](https://docs.docker.com/engine/network/packet-filtering-firewalls/#integration-with-firewalld).

## Setup Firewalld

To install and enable firewalld on a Debian distro:
```bash
sudo apt update
sudo apt install firewalld 
sudo systemctl enable firewalld
```
After that, reboot the system.

## Configure Zones

 Docker will create a new zone `docker` and add all bridge network interfaces to that zone and create a rule called "docker-forwarding" which allows ingress traffic to the containers. For our scenario it's not a problem as we just want to isolate the traffic coming from the containers.

To achieve that, we first set the docker zone from `Accept` to `Drop` to drop all traffic coming from the containers by default.
```bash
sudo firewall-cmd --zone docker --set-target DROP --permanent
sudo firewall-cmd --reload
sudo firewall-cmd --info-zone docker
docker (active)
  target: DROP
  ingress-priority: 0
  egress-priority: 0
  icmp-block-inversion: no
  interfaces: br-7a35a1c3d71e br-b1a5a369be8e br-cd1196165875 docker0
  sources: 
  services: 
  ports: 
  protocols: 
  forward: yes
  masquerade: no
  forward-ports: 
  source-ports: 
  icmp-blocks: 
  rich rules:
```

It makes sense to set the default zone on the Docker host to a restrictive zone like `dmz` which allows only `ssh` connections to the host.

Use `sudo firewall-cmd --list-all-zones` to list all predefined zones and their settings.
```bash
firewall-cmd --zone dmz --list-all
dmz (active)
  target: default
  ingress-priority: 0
  egress-priority: 0
  icmp-block-inversion: no
  interfaces: 
  sources: 
  services: ssh
  ports: 
  protocols: 
  forward: yes
  masquerade: no
  forward-ports: 
  source-ports: 
  icmp-blocks: 
  rich rules:

firewall-cmd --set-default-zone dmz
firewall-cmd --get-default-zone
```

## Test & Logs

To test this and check the logs for blocked connections, first enable the logging of denied connections:
```bash
sudo firewall-cmd --set-log-denied all
sudo firewall-cmd --get-log-denied
```

Then we can try to ping a host on the local network or try getting to a internet site:
```bash
docker run -it --rm nicolaka/netshoot

ping 192.168.1.1

nmap docker.com -p 443
Starting Nmap 7.97 ( https://nmap.org ) at 2025-12-13 20:10 +0000
Nmap scan report for docker.com (23.185.0.4)
Host is up (0.013s latency).
Other addresses for docker.com (not scanned): 2620:12a:8000::4 2620:12a:8001::4

PORT    STATE    SERVICE
443/tcp filtered https

Nmap done: 1 IP address (1 host up) scanned in 0.72 seconds
```

In another console window we can check the live logs:
```bash
sudo journalctl -f
Nov 30 18:18:15 host kernel: filter_FWD_docker_DROP: IN=docker0 OUT=eno1 MAC=1e:02:0e:b9:de:22:2a:ce:54:05:c6:3f:08:00 SRC=172.17.0.2 DST=192.168.1.1 LEN=84 TOS=0x00 PREC=0x00 TTL=63 ID=20238 DF PROTO=ICMP TYPE=8 CODE=0 ID=26 SEQ=10
```

## Allow traffic with Policies

Now if the container requires certain traffic, e.g. `HTTP` traffic to the internet, simply create a policy that allows it:
```bash
policy="allow-internet"
sudo firewall-cmd --new-policy $policy --permanent
sudo firewall-cmd --policy $policy --permanent --add-ingress-zone docker
sudo firewall-cmd --policy $policy --permanent --add-egress-zone dmz
sudo firewall-cmd --policy $policy --permanent --add-rich-rule='rule family="ipv4" destination NOT address="192.168.0.0/16" service name="http" accept'
sudo firewall-cmd --policy $policy --permanent --add-rich-rule='rule family="ipv4" destination NOT address="192.168.0.0/16" service name="https" accept'
sudo firewall-cmd --reload
sudo firewall-cmd --info-policy $policy
```

> The `target` is the action that is used for every packet that doesn't match any rule (port, service, etc.). `Continue` means that packets that are not `http` or `https` are processed by the `docker` zone which has target `drop` and therefore will be blocked.
{: .prompt-info}

> The two rich rules are required to say: allow only traffic that doesn't have a target address in the local network.
{: .prompt-info}
