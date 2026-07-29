---
layout: post
title:  "Azure Blob SFTP: what Bicep can't do for you"
date:   2026-07-29
categories: azure
---
First of all, to enable SFTP on Azure Blob Storage, it requires hierarchical namespace (`isHnsEnabled: true`, GPv2 or premium block blob). And SFTP bills hourly whether or not you are actually transferring anything.

Now the real fun. SFTP local users are scoped to **containers** — you pick the containers a user can reach and the permission letters they get on each one (Read `r`, Write `w`, List `l`, Delete `d`, Create `c`, Modify Ownership `o`, Modify Permissions `p`). Those permissions apply to every directory and subdirectory in the container. There is no container permission that means "only this sub-folder". For that you need POSIX ACLs.

The users I created needed different access to different folders within the same container, and it took me hours to figure out that I couldn't do any of it in Bicep.

Two separate reasons, and I kept conflating them at first. Bicep/ARM is the management plane and doesn't set ACLs at all — ACLs live on the data plane. And the data plane API (Data Lake Gen2) *can* set ACLs on a directory, but it can't point one at a local user. Straight from [the docs](https://learn.microsoft.com/en-us/azure/storage/blobs/secure-file-transfer-protocol-support#modifying-acls-with-an-sftp-client):

>"only local users can change the ID of the owning user or owning group to a local user ID. You can't yet reference a local user ID in an ACL by using an Azure tool or SDK."

So the ACL has to be applied from inside an SFTP session. Fine. But there's a second trap that decides how you write your Bicep, and this one is the actual reason my template looks weird:

>"ACLs are evaluated only if the local user doesn't have the necessary container permissions to perform an operation. Because of the way that access permissions are evaluated by the system, you can't use an ACL to restrict access that has already been granted by container-level permissions."

Container permissions win. If you grant `rw` on the container and then try to fence the user into one sub-folder with an ACL, the ACL is never even consulted. The only way to make ACLs matter is to *not* grant the container permission in the first place. Thanks Microsoft!

So while I was able to provision the storage account, containers and local users in Bicep, I had to set the container permissions for 2 users to the bare minimum. Note you can't strip them to nothing — a local user needs at least one container permission, or an ACL on its home directory, otherwise the connection itself fails. Then I logged in as an admin user (which needs `o` and `p` on the container to be allowed to do this at all) and ran `chmod` and `chown` one by one until the permissions were exactly right. It was a real pain.

One more landmine while you're in there: SFTP only understands the plain POSIX `user::`, `group::` and `other::` entries. If any directory in the access path — including the user's home directory — has a named-user entry or a default ACL on it, SFTP operations fail with `Permission denied` even when the user genuinely has the right permissions. So don't reach for named users to solve this. They'll break the thing you just fixed.

The users can be authenticated using either a password or a public key. The public key is a clear winner here in my opinion — Azure generates the password for you, it's shown exactly once, and you can never retrieve it again.

You can also link a private endpoint to your storage account or just leave it open to all public network, or only open to selected vNet/IP using a white list.

One of these users is used by a function app from a separate vNet which failed to resolve the private endpoint of this storage account. It's because the vNet that hosts the function app has a custom DNS server from AD. It's the wrong setup but I cannot change it. So instead of using FQDN of my storage endpoint, I had to use its public IP and allow the function app vNet to reach my storage via Azure backbone. For that vNet rule to actually match the traffic, the function app needs `vnetRouteAllEnabled` on and the `Microsoft.Storage` service endpoint enabled on the integration subnet.

It works, but it's ugly, and it's borrowed time — Azure [does not guarantee a static IP](https://learn.microsoft.com/en-us/azure/storage/blobs/secure-file-transfer-protocol-known-issues#networking) for a storage account, so that hardcoded address can change out from under me.

## What I should have done about the DNS

Writing this up made me go and look properly, and there were more ways out than I thought. All of them assume the `privatelink.blob.core.windows.net` private DNS zone exists and is linked to the vNet — that part is non-negotiable. Once DNS works you connect as `myaccount.myuser@myaccount.privatelink.blob.core.windows.net` and the private endpoint does its job.

**Conditional forwarder on the AD DNS server.** The textbook fix: forward `privatelink.blob.core.windows.net` to `168.63.129.16`, the Azure recursive resolver. One resolution path, nothing app-specific, works for everything else in the vNet too. It just needs someone with rights on the AD DNS server to do it, which is exactly the thing I didn't have.

**Override DNS on the function app itself.** This is the one I wish I'd known. An app can ignore the vNet's DNS servers and use its own list of up to five, set via the `dnsConfiguration.dnsServers` site property:

```
az resource update -g <group> -n <app> --resource-type "Microsoft.Web/sites" \
  --set properties.dnsConfiguration.dnsServers="['168.63.129.16']"
```

The older `WEBSITE_DNS_SERVER` app setting does the same thing, but site properties win when both exist and Microsoft recommends them — they're auditable with Azure Policy and validated when you set them. No AD change, scoped to one app.

The catch is that it *replaces* the inherited list, so that function app stops resolving internal AD names. Fine for mine, which only talks to storage. Not fine if your app also calls something on the corporate network.

And a trap worth knowing: `WEBSITE_DNS_ALT_SERVER` looks like the answer here, because it *appends* Azure DNS to the inherited servers instead of replacing them. It isn't. The app queries every configured server, but takes the first response it gets back inside the timeout — and a wrong answer is still a response. AD DNS forwarding to the public internet will happily resolve the storage FQDN to its public IP, return that, and the alternative server never gets a look in. Appending only rescues you when the first server doesn't answer at all.

**Azure DNS Private Resolver.** If you can change the vNet's DNS setting but not the AD server itself, this inverts the problem. Point the vNet at the resolver's inbound endpoint, then use a forwarding ruleset to send the AD zones back to the AD DNS servers and let everything else resolve through Azure DNS and the linked private zones. AD stays authoritative for what it owns and you stop fighting it for everything else. It's a real resource with a real bill, so it's overkill for one storage account, but it's the right answer if you have a fleet of private endpoints in the same situation.

**Just add the A record.** Least elegant, occasionally the only one that gets approved: have the AD team add a single A record for `<account>.privatelink.blob.core.windows.net` pointing at the private endpoint's private IP. A record addition is a much smaller ask than a forwarder in most shops, and private endpoint IPs don't move once allocated. It doesn't scale and nobody will remember it exists in a year, but it's honest work.

Ranked by what I'd actually reach for: conditional forwarder if I can get it, app-level DNS override if I can't, private resolver if this keeps happening, A record if I'm out of options and out of patience.

## References

- [SFTP support for Azure Blob Storage](https://learn.microsoft.com/en-us/azure/storage/blobs/secure-file-transfer-protocol-support) — the permission model, how ACLs are evaluated against container permissions, and the home directory rules. Both quotes above come from here.
- [Authorize SFTP access to Azure Blob Storage](https://learn.microsoft.com/en-us/azure/storage/blobs/secure-file-transfer-protocol-support-authorize-access) — creating local users and assigning container permissions in the portal, PowerShell and CLI. Also where the "Allow ACL authorization" toggle lives, which you have to turn on per user before any of this works.
- [Connect to Azure Blob Storage from an SFTP client](https://learn.microsoft.com/en-us/azure/storage/blobs/secure-file-transfer-protocol-support-connect) — worked `chmod` / `chown` / `chgrp` examples, and the connection string formats including the private endpoint one (`myaccount.myuser@myaccount.privatelink.blob.core.windows.net`).
- [Limitations and known issues with SFTP](https://learn.microsoft.com/en-us/azure/storage/blobs/secure-file-transfer-protocol-known-issues) — the named-user/default ACL trap, no static IPs, the 500 GB file cap, and the 2 minute idle timeout.
- [Access control lists (ACLs) in Azure Data Lake Storage](https://learn.microsoft.com/en-us/azure/storage/blobs/data-lake-storage-access-control) — the POSIX model underneath all of this. Read the bit about needing Execute on *every* parent directory down to the file; it's the thing that catches everyone out.
- [Azure quickstart: storage account with SFTP](https://github.com/Azure/azure-quickstart-templates/tree/master/quickstarts/microsoft.storage/storage-sftp) — official Bicep and ARM sample that provisions the account, container and local users.
- [Azure Storage firewall rules and network access](https://learn.microsoft.com/en-us/azure/storage/common/storage-network-security) — vNet rules, service endpoints and IP rules. Note that once a subnet has a service endpoint, its traffic stops using a public source IP, so any IP rule covering that subnet quietly stops applying.
- [Name resolution (DNS) in App Service](https://learn.microsoft.com/en-us/azure/app-service/overview-name-resolution) — the `dnsConfiguration.dnsServers` override, the `WEBSITE_DNS_*` app settings, and how App Service picks a winner when several DNS servers answer.
- [Private endpoint DNS configuration](https://learn.microsoft.com/en-us/azure/private-link/private-endpoint-dns) — the zone names and the forwarding patterns for custom DNS. The App Service docs tell you to read this one carefully if you have custom DNS servers and private endpoints, and they're right.
- [Azure DNS Private Resolver overview](https://learn.microsoft.com/en-us/azure/dns/dns-private-resolver-overview) and [endpoints and rulesets](https://learn.microsoft.com/en-us/azure/dns/private-resolver-endpoints-rulesets) — inbound endpoints and conditional forwarding rules, if you want to fix this properly rather than per app.
