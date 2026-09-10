---
layout: post
title:  "Firewall what should the destination IP be for a cross-tenant server"
date:   2026-08-12
categories: azure
---
context
a vm in spoke vnet needs to talk to a client's database in AWS. there's a VPN connection between this client's AWS and my Azure tenant, along with 2 NAT rules (ingress and egress). So when my VM sends a request to client's DB server (which is likely behind a gateway that I cannot see because the IP provided by the client is 172.30.0.1), it reaches VPN gateway gets its source IP translated to something like 172.30.0.5.

I had no idea what route maps were. I was only responsible for creating the firewall policy to allow the traffice between the source VM and client's DB server in AWS. I was already given the soure and destination IPs, and port number for the firewall policy. So my task was really easy. But I was curious about the strange destination IP 10.30.0.1 which seems to be a reserved address. It looked wrong to me at the time. Little did I know then I was about to go into a rabbit hole of azure networking.

Through this exploration and self-learning, I got much better understanding of Azure networking:
- hub and spoke
- vWAN
- transitive routing
- mesh network
- NAT
- VPN
- BGP

some of these were things I kept hearing throughout my career but never needed to understand how they really work. for example, I know NAT and VPN but I had no idea how they actually work. I heard of BGP but I had no idea what it really was. I knew hub and spoke but only on a superficial level.

vWAN is mesh network of vWAN hubs.

what's azure vwan

what are route maps
inbound and outbound rule, direction semantics, it's relative to the hub. 

mesh network
hubs connected to each other without a central hub, transitive routing

routing

firewall
