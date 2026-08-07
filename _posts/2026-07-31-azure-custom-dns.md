---
layout: post
title:  "Azure custom DNS: when the textbook fix returns NXDOMAIN"
date:   2026-07-31
categories: azure
---
Two days ago I wrote that the textbook fix for a function app that can't resolve a private endpoint is a conditional forwarder on the AD DNS server pointing at `168.63.129.16`. I still think that's right. But I've since watched that exact configuration return `NXDOMAIN` for three days, and the reason is a thing I had completely wrong in my head: **`168.63.129.16` is not a resolver. It's a resolver *per vNet*, and which vNet it answers from is decided by where the machine asking the question lives - not by where the private endpoint lives.**

Four vNets, which I'll name after their jobs because I confused myself badly enough drawing this on a whiteboard:

| vNet | Job | On the vWAN hub? |
|---|---|---|
| `vnet-app` | function app, regional vNet integration | spoke |
| `vnet-storage` | storage account private endpoint | spoke |
| `vnet-dns` | Azure DNS Private Resolver | spoke |
| `vnet-identity` | AD DS domain controllers (Azure VMs) | **no** |

Azure Firewall is integrated with the vWAN hub, so spoke-to-spoke traffic is filtered. The storage account is `saplatform.blob.core.windows.net` and its private endpoint sits in `vnet-storage`. The `privatelink.blob.core.windows.net` private DNS zone is linked to `vnet-dns` and contains the `saplatform` A record. `vnet-storage` uses the resolver's inbound endpoint as its custom DNS. So far, textbook.

-

`vnet-app` is where it gets interesting. Its custom DNS is the domain controllers, which is mandatory  the apps there need to resolve internal AD names and I can't change that. Those DCs are Azure VMs, and they live in `vnet-identity`, which isn't attached to the vWAN hub and spoke at all. They have a conditional forwarder for `windows.net` pointing at `168.63.129.16`.

The function app has no site-level DNS override, so it inherits the DCs from `vnet-app`. It needs to reach the storage account over SFTP. Firewall rules from the integration subnet to the private endpoint subnet were in place. And every lookup came back `NXDOMAIN`.

## NXDOMAIN is the clue, not the problem

The first thing worth saying is that `NXDOMAIN` was the most useful piece of information I had, and I ignored it for a day and a half while I re-read firewall rules.

A DNS query that can't get anywhere times out, or comes back `SERVFAIL`. `NXDOMAIN` is different: it means a server that considers itself **authoritative** for that name answered, deliberately, that the name does not exist. Something was working. Something was answering. It was just answering "no".

That single fact rules out most of what I'd been staring at. Firewall rules on port 53 - fine, or I'd have had a timeout. Routing to the DCs - fine. The conditional forwarder itself - being used, or the DCs would have gone to their general forwarders and I'd have seen something else entirely.

It also rules out the failure I *expected* to find. If a `privatelink` zone simply isn't linked where it needs to be, you don't get `NXDOMAIN` - you get the storage account's **public** IP. `saplatform.blob.core.windows.net` publicly CNAMEs to `saplatform.privatelink.blob.core.windows.net`, and Microsoft resolves that publicly to the real front end. A missing link gives you a wrong answer, not no answer. I was getting no answer, so the problem was the opposite: somebody was authoritative and empty.

The other half of the clue was in the partial resolution. The CNAME resolved fine. `saplatform.blob.core.windows.net` → `saplatform.privatelink.blob.core.windows.net`, and then death. So the break wasn't on the name I was asking about. It was on the name that answer pointed me at.

## Two zones, one name

There were **two** private DNS zones called `privatelink.blob.core.windows.net` in the tenant.

| Zone | Linked to | Has the `saplatform` record |
|---|---|---|
| the resolver's zone | `vnet-dns` | yes |
| the identity zone | `vnet-identity` | **no** |

The identity zone had been created for that vNet's own private endpoints, long before any of this, by people solving a different problem. It's a perfectly reasonable zone. It just happens to have the same name.

And that's the whole bug, because of two Azure behaviours that are individually documented and jointly vicious:

**A linked private DNS zone makes Azure DNS authoritative for that entire namespace, for queries from that vNet.** Not "authoritative for the records in it" - authoritative for the *name*. A lookup for something in that namespace that isn't in the zone returns `NXDOMAIN`. It does not fall through to public DNS, and it does not go looking for another zone with the same name. That's by design; it's how split-horizon DNS is supposed to work. It's also why a half-populated private zone is more dangerous than no private zone at all.

**`168.63.129.16` resolves in the context of the vNet of the machine that queries it.** It's a fixed virtual IP that exists identically in every vNet, and every time I'd thought about it I'd thought of it as "the Azure resolver", singular. It isn't. Ask it from `vnet-identity` and you get that vNet's view of the world, including its linked private zones. The private endpoint being in `vnet-storage` is irrelevant. The record being in the resolver's zone is irrelevant. The DC is in `vnet-identity`, so it got `vnet-identity`'s answer.

Putting those together, the actual trace:

```
function app (vnet-app)
  → DC in vnet-identity:  saplatform.blob.core.windows.net
  → conditional forwarder "windows.net" → 168.63.129.16
                                          [resolves in vnet-identity's context]
  → public CNAME:  saplatform.privatelink.blob.core.windows.net
  → vnet-identity is linked to the identity zone
  → Azure DNS is AUTHORITATIVE for privatelink.blob.core.windows.net here
  → no saplatform record
  → NXDOMAIN
```

The conditional forwarder was doing precisely what I told it to. It forwarded to Azure DNS. Azure DNS answered honestly. Every component behaved correctly and the result was still a broken app.

To confirm it, from a DC:

```powershell
Get-DnsServerZone -Name windows.net | Select-Object ZoneName, ZoneType, MasterServers

Resolve-DnsName saplatform.privatelink.blob.core.windows.net -Server 168.63.129.16
Resolve-DnsName saplatform.privatelink.blob.core.windows.net -Server <inbound-endpoint-IP>
```

`MasterServers` showed `168.63.129.16`. First lookup, `NXDOMAIN`. Second, the private IP. That's the entire diagnosis in two commands, and I'd have got there on day one if I'd trusted what `NXDOMAIN` was telling me.

## The constraint that decides your fix

The obvious repair is "link the good zone to `vnet-identity` as well". You can't. **A vNet can be linked to only one private DNS zone of a given name.** The link operation fails on the name collision. So you're choosing between the two zones, not combining them, and that constraint is what makes this annoying rather than trivial.

What's theoretically available:

**Repoint the conditional forwarder at the private resolver's inbound endpoint.** Instead of `168.63.129.16`, forward `windows.net` to the inbound endpoint in `vnet-dns`. Resolution then happens in that vNet's context, where the right zone is linked and the record exists, and the identity zone never enters the picture. This is the surgical fix and it's what the architecture was clearly meant to do - there's a private resolver sitting right there and the DCs are the only thing not using it.

**Take AD DS out of the path for `vnet-app` entirely.** Point its custom DNS at the resolver's inbound endpoint, and add a forwarding ruleset with an outbound endpoint sending the AD domain suffix back to the DCs. The app vNet gets private zone resolution natively and AD resolution via the ruleset, and the conditional forwarder stops being load-bearing.

**Add the A record to the identity zone by hand.** Point `saplatform.privatelink.blob.core.windows.net` at the private endpoint's IP, in the zone that's already authoritative for the query. Sixty seconds. Fixes this storage account and nothing else.

And what's actually available: the third one.

I don't have access to the domain controllers. I didn't have it [last time either]({% post_url 2026-07-29-azure-sftp-storage %}) - that post ends with me recommending a conditional forwarder and noting it "just needs someone with rights on the AD DNS server to do it, which is exactly the thing I didn't have". Two posts in a row now where the correct fix is sitting behind somebody else's RBAC. That's not an Azure problem and there's no configuration that solves it.

So I added the record. It's the option I'd have sniffed at a week ago, and in context it's fine, for one specific reason: it has an end date. `vnet-dns` is already the central hub DNS - it has the full set of `privatelink.*` zones linked to it, and we're migrating every vNet onto it, which is the [Private Link and DNS integration at scale](https://learn.microsoft.com/en-us/azure/cloud-adoption-framework/ready/azure-best-practices/private-link-and-dns-integration-at-scale) pattern. Once there's exactly one zone per name and everything links to it, there's nothing left to collide. When `vnet-identity` gets relinked, the duplicate zone goes away and my hand-written record goes with it, because the central zone already has the same record created automatically by the private endpoint's zone group. The stopgap deletes itself at migration.

The risk I'm carrying until then is drift. If that private endpoint is ever recreated it gets a new IP, the auto-populated record in the central zone follows it, and my manual copy in the identity zone quietly doesn't. Nothing alerts on that. It fails the same way this did - `NXDOMAIN`, or worse, a stale IP that times out - and the next person has to rediscover this whole post to work out why.

## Go and check your own tenant

Nothing that went wrong here was specific to blob storage. Two zones with the same name, linked to different vNets, one of them missing a record - that works just as well for `privatelink.database.windows.net`, `privatelink.vaultcore.azure.net`, or any other private endpoint zone. So the question isn't whether this storage account is fixed. It's how many more of these you already have.

They're invisible until someone creates a private endpoint for that particular service and gets an `NXDOMAIN` nobody can explain. Until then a duplicate zone name looks like nothing at all.

And if you're partway through centralising your private DNS, as we are, duplicates aren't a possibility - they're guaranteed. You have the new central zones and the old per-vNet ones at the same time, by definition, for however long the migration takes. Which is always longer than planned.

```powershell
Get-AzPrivateDnsZone | ForEach-Object {
  Get-AzPrivateDnsVirtualNetworkLink -ZoneName $_.Name -ResourceGroupName $_.ResourceGroupName
} | Select-Object ZoneName, Name, VirtualNetworkId | Sort-Object ZoneName
```

Any zone name appearing twice is worth knowing about, but the ones to chase first are the vNets that **host DNS servers** - domain controllers especially. A vNet linked to a `privatelink` zone imposes its DNS context on every client that queries a resolver living there, including clients in completely unrelated vNets that have no relationship with it at all. That's the bit that got me, and it's the bit worth taking away if you take away nothing else: the function app in `vnet-app` was subject to `vnet-identity`'s private DNS zone links, and there is nothing in `vnet-app`'s configuration, or the storage account's, or the private endpoint's, that would ever tell you so. Your effective DNS is decided by the vNet of whichever machine ends up answering the question, and that machine can be somewhere you've never looked.

Prioritise those vNets in the migration for the same reason. A spoke on the old zone only breaks itself. A DNS server on the old zone breaks everything downstream of it.

## Still on the list

DNS resolving isn't the same as SFTP connecting, and I haven't fully proved out the data path yet. Private endpoints don't honour route tables unless `privateEndpointNetworkPolicies` is enabled on their subnet, which in a vWAN topology with a hub firewall is a good way to get asymmetric routing - the firewall is stateful, sees one side of the flow, and drops it. If that bites, it'll present as a hang or a reset rather than a name resolution error, which at least makes it easy to tell apart from this one. Post to follow if it does.

## References

- [What is IP address 168.63.129.16?](https://learn.microsoft.com/en-us/azure/virtual-network/what-is-ip-address-168-63-129-16) - the one to read if, like me, you thought of it as a single global resolver. It's a per-vNet virtual IP and it answers from the caller's vNet.
- [Azure Private DNS overview](https://learn.microsoft.com/en-us/azure/dns/private-dns-overview) and [virtual network links](https://learn.microsoft.com/en-us/azure/dns/private-dns-virtual-network-links) - the authoritative-for-the-namespace behaviour and the one-zone-per-name-per-vNet link constraint that decides which fix you get to use.
- [Private endpoint DNS configuration](https://learn.microsoft.com/en-us/azure/private-link/private-endpoint-dns) - the zone names per service, and the CNAME chain from the public name into `privatelink`. Understanding that chain is what turns "NXDOMAIN" from a mystery into a location.
- [Private Link and DNS integration at scale](https://learn.microsoft.com/en-us/azure/cloud-adoption-framework/ready/azure-best-practices/private-link-and-dns-integration-at-scale) - the CAF pattern for a single central zone per name. Written for greenfield, but it's the target state to aim a migration at.
- [Azure DNS Private Resolver](https://learn.microsoft.com/en-us/azure/dns/dns-private-resolver-overview) and [endpoints and rulesets](https://learn.microsoft.com/en-us/azure/dns/private-resolver-endpoints-rulesets) - inbound endpoints for forwarding *into* Azure, outbound endpoints and rulesets for forwarding back out to AD.
- [Name resolution in App Service](https://learn.microsoft.com/en-us/azure/app-service/overview-name-resolution) - how an integrated app inherits vNet DNS servers, and the site-level override if you need to escape them.
- [Azure Blob SFTP: what Bicep can't do for you]({% post_url 2026-07-29-azure-sftp-storage %}) - the previous instalment, in which I confidently recommend the conditional forwarder that caused this.
