---
layout: post
title:  "Setting up Traffic Manager in front of Application Gateways"
date:   2026-07-26 14:03:42 +1000
categories: azure
---
## Step 1 Deploy and Configure Application Gateways

- Deploy and Configure Application GatewaysProvision your Application Gateway instances in different target regions.
- Assign a Public IP address to the frontend configuration of each Application Gateway.
- Ensure each regional gateway is fully operational and servicing local traffic independently via its respective public IP.

## Step 2: Create a Traffic Manager Profile

- Go to the Azure Portal, search for and select Traffic Manager profiles, and click Create.
- Enter a unique Name for your global DNS profile (e.g., app-traffic-manager.trafficmanager.net).
- Select a Routing method (such as Performance for lowest latency, Priority for active-failover, or Weighted).
- Choose your target Subscription and Resource Group, then click Create.

## Step 3: Add Application Gateways as Endpoints

- Open your newly created Traffic Manager profile.
- Navigate to Settings > Endpoints and click Add.Set Type to Azure endpoint.
- Provide a descriptive Name for the regional endpoint.
- Select Public IP address for the Target resource type.
- Choose the Public IP address belonging to your regional Application Gateway.
- Repeat this process to add the public IPs for your other regional Application Gateways.

## Step 4: Verify and Test

- Save the endpoint configurations and note the DNS name of your Traffic Manager profile.
- Open a web browser and navigate to the Traffic Manager DNS name to verify it correctly routes traffic to the active regional Application Gateway.
