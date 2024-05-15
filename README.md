# AS-path Prepends with Azure VMware Solution

Goal of this article is to explain As-path prepending behaviors when using ExpressRoute and Azure hubs with Azure VMware Solution and why Public ASNs should be used to prepend towards AVS.

## Scenario1: Single ExpressRoute Circuit. Use one link as primary and second for failover using prepends with private ASN
<img width="574" alt="image" src="https://github.com/shrnair/AS-path-Prepends-with-Azure-VMware-Solution/assets/47249875/3fd704e5-7c7b-4511-a9db-b85a35a3a74f">

I have a single ExpressRoute circuit with two links. On-prem prefix 10.61.0.0/24 is being advertised from both links of ExpressRoute with different As-path length. Path a advertises with shorter As-path and Path b advertises with longer As-path 

In a single ExpressRoute circuit you have two physical connection. 
MSEE1 has BGP session with R1 and MSEE2 has BGP session with R2

What MSEE1 sees?

<img width="472" alt="image" src="https://github.com/shrnair/AS-path-Prepends-with-Azure-VMware-Solution/assets/47249875/e860038c-9e1e-4275-b1c9-54443d46dd52">


What MSEE2 sees?

<img width="472" alt="image" src="https://github.com/shrnair/AS-path-Prepends-with-Azure-VMware-Solution/assets/47249875/a402416c-5471-4006-9e74-bf01dc74bb59">


How does DMSEE1 sees the route?

DMSEE1 sees the prefix 10.61.0.0/24 without prepend

<img width="427" alt="image" src="https://github.com/shrnair/AS-path-Prepends-with-Azure-VMware-Solution/assets/47249875/2d88461c-a542-42cf-a050-6d593f8f2aa9">


How does DMSEE2 sees the route with prepend?

<img width="472" alt="image" src="https://github.com/shrnair/AS-path-Prepends-with-Azure-VMware-Solution/assets/47249875/be9eaafa-84a9-4b12-baf1-cf10ac56718a">



How does DMSEE advertise the route to AVS Edge?

<img width="427" alt="image" src="https://github.com/shrnair/AS-path-Prepends-with-Azure-VMware-Solution/assets/47249875/da3d9281-2331-48f2-914f-2ad09006a9d7">





All the private ASNs are stripped off by both DMSEEs before advertising them to AVS edge leaf. AVS Edge leaf will receive the prefix 10.61.0.0/24 from both DMSEEs with Aspath of 12076
Hence cannot differentiate which DMSEE is a better path as both have same Aspath length and will pick any . Hence with a single ExpressRoute circuit path preference is not possible.

## Solution:

1. Use Public ASN for prepends as DMSEEs will not strip Public AS and now AVS leaf will be able to identify the best path based on the As-path prepend
   Here is an example advertised route where Public ASN is maintained

<img width="427" alt="image" src="https://github.com/shrnair/AS-path-Prepends-with-Azure-VMware-Solution/assets/47249875/60cdf83c-2f52-46d1-85d0-41ac54e2d9d2">


2. Contact Azure Support who will help implement prepend on the inbound direction of AVS leaf to achieve path preference. This will not be visible to customer and every time any change needs to be done on prepends you need to contact support.

## Scenario 2: Influencing path preference with two circuits
In this example one ExpressRoute circuit is in Dallas and another one in Chicago. We want to use Dallas as the primary path and Chicago as the Secondary/failover path.

<img width="700" alt="image" src="https://github.com/shrnair/AS-path-Prepends-with-Azure-VMware-Solution/assets/47249875/cf2dab9b-6e87-4249-83cd-c1d2af870115">


All path a, b, c and d will advertise on-prem prefix 10.61.0.0/24.
Path a and b will advertise 10.61.0.0/24 without any additional prepends. I will only put sample outputs from MSEE1. Both MSEE1 and MSEE2 routes would be identical.

What MSEE1 sees?

<img width="427" alt="image" src="https://github.com/shrnair/AS-path-Prepends-with-Azure-VMware-Solution/assets/47249875/6fbde62c-0168-4b69-9d8d-3bac23f2691d">


Path c and d will advertise 10.61.0.0/24 with prepends.

What MSEE3 sees?

<img width="427" alt="image" src="https://github.com/shrnair/AS-path-Prepends-with-Azure-VMware-Solution/assets/47249875/3786991b-ea04-4d57-893c-5af050787ec8">


What DMSEE1 sees?

The BGP routing table has both routes from path a and c. But Path a is preferred as it is shorter 

<img width="460" alt="image" src="https://github.com/shrnair/AS-path-Prepends-with-Azure-VMware-Solution/assets/47249875/db712039-c804-43d9-b6ed-76b8dbbfa31d">


What DMSEE2 sees?

The BGP routing table has both routes from path b and d. But Path b is preferred as it is shorter

<img width="460" alt="image" src="https://github.com/shrnair/AS-path-Prepends-with-Azure-VMware-Solution/assets/47249875/218b989e-76e4-4660-a1ed-0d32f72fe6c3">

How is the route advertised to AVS Edge?

<img width="460" alt="image" src="https://github.com/shrnair/AS-path-Prepends-with-Azure-VMware-Solution/assets/47249875/c08a6c7a-e99f-412a-9fa9-a1764dc193c8">


The ASNs are still stripped of by the DMSEE. But since both DMSEE have identical routes and both identify Dallas(a and b) as shorter path. Hence the traffic from AVS to On-Prem will always route through Dallas circuit. Unless the routes are lost or the circuit is down.

## Problem:

While everything will route as expected when the all links are up and stable. But lets assume primary link between Onprem and MSEE1 is down and onprem routes are not advertised via MSEE1 to DMSEE1. In this scenario when AVS tries to reach Onprem since it only sees ASN 12076 the traffic mifgt end up on DMSEE1 which has the path via Chicago and not Dallas. As you can see even though the secondary link from Dallas is up the possibility that Chicago path is used is still there.

## Solution:

1. Use Public ASN for prepends as DMSEEs will not strip Public AS and now AVS leaf will be able to identify the best path based on the As-path prepend
   Here is an example advertised route where Public ASN is maintained

<img width="427" alt="image" src="https://github.com/shrnair/AS-path-Prepends-with-Azure-VMware-Solution/assets/47249875/60cdf83c-2f52-46d1-85d0-41ac54e2d9d2">


2. Contact Azure Support who will help implement prepend on the inbound direction of AVS leaf to achieve path preference. This will not be visible to customer and every time any change needs to be done on prepends you need to contact support.

## Scenario 3:Need internet breakout from AVS to be routed through an Azure hub. Need to have a backup Internet path through another hub.

There are two Hub with two NVAs each(I used CSR in my setup) advertising 0/0 using ARS and have next hop as the ILB IP1 and ILB IP2.

<img width="647" alt="image" src="https://github.com/shrnair/AS-path-Prepends-with-Azure-VMware-Solution/assets/47249875/4269dcfb-b2f9-49d5-a685-6e0a3772bd8a">

What DMSEE1 sees?

It receives 0/0 from both Vnet-prod and Vnet-dev.
Vnet-prod is the preferred path due to shorter As-path length

<img width="500" alt="image" src="https://github.com/shrnair/AS-path-Prepends-with-Azure-VMware-Solution/assets/47249875/cd78acb6-e185-4c9d-b6ce-939b4f265c90">


What DMSEE2 sees?

It is identical to DMSEE1 routes

<img width="500" alt="image" src="https://github.com/shrnair/AS-path-Prepends-with-Azure-VMware-Solution/assets/47249875/97ff984c-6778-45d8-b3e7-610d7907e5f1">

What DMSEE advertises to AVS edge?

<img width="427" alt="image" src="https://github.com/shrnair/AS-path-Prepends-with-Azure-VMware-Solution/assets/47249875/ceacb69f-feb8-49f1-862e-8f39fc4ca2a9">

The ASNs are still stripped of by the DMSEE. But since both DMSEE have identical routes and both identify path a as shorter path due to As-path length. Hence the traffic from AVS to Internet will always route through Vnet-Prod. Unless the routes are lost from Vnet-Prod.








