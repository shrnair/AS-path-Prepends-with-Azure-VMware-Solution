# AS-path Prepends with Azure VMware Solution

Goal of this article is to explain As-path prepending behaviors when using ExpressRoute and Azure hubs with Azure VMware Solution

## Scenario1: Single ExpressRoute Circuit. Use one link as primary and second for failover using prepends with private ASN
<img width="574" alt="image" src="https://github.com/shrnair/AS-path-Prepends-with-Azure-VMware-Solution/assets/47249875/833faa0d-9b90-4f54-9c57-b4483e446f3d">

I have a single ExpressRoute circuit with two links. On-prem prefix 10.61.0.0/24 is being advertised from both links of ExpressRoute with different As-path length. Path a advertises with shorter As-path and Path b advertises with longer As-path 

In a single ExpressRoute circuit you have two physical connection. 
MSEE1 has BGP session with R1 and MSEE2 has BGP session with R2

What MSEE1 sees?

<img width="376" alt="image" src="https://github.com/shrnair/AS-path-Prepends-with-Azure-VMware-Solution/assets/47249875/acd5350a-98f1-46b2-8609-cef782276579">


What MSEE2 sees?

<img width="472" alt="image" src="https://github.com/shrnair/AS-path-Prepends-with-Azure-VMware-Solution/assets/47249875/094b0d70-1a76-4f01-9675-1817c07d0673">

How does DMSEE1 sees the route?

DMSEE1 sees the prefix 10.61.0.0/24 without prepend

<img width="361" alt="image" src="https://github.com/shrnair/AS-path-Prepends-with-Azure-VMware-Solution/assets/47249875/ef7c31b7-0d94-42db-85d4-745759e95390">

How does DMSEE2 sees the route with prepend?

<img width="427" alt="image" src="https://github.com/shrnair/AS-path-Prepends-with-Azure-VMware-Solution/assets/47249875/ad505912-91ac-4625-ac8f-9fd19ff69856">


How does DMSEE advertise the route to AVS Edge?

<img width="475" alt="image" src="https://github.com/shrnair/AS-path-Prepends-with-Azure-VMware-Solution/assets/47249875/5572f8a8-bc63-4798-bd8b-fdd306d8bade">




All the private ASNs are stripped off by both DMSEEs before advertising them to AVS edge leaf. AVS Edge leaf will receive the prefix 10.61.0.0/24 from both DMSEEs with Aspath of 12076
Hence cannot differentiate which DMSEE is a better path as both have same Aspath length and will pick any . Hence with a single ExpressRoute circuit path preference is not possible.

## Solution:

1. Use Public ASN for prepends as DMSEEs will not strip Public AS and now AVS leaf will be able to identify the best path based on the As-path prepend
   Here is an example advertised route where Public ASN is maintained

<img width="482" alt="image" src="https://github.com/shrnair/AS-path-Prepends-with-Azure-VMware-Solution/assets/47249875/d29047c4-039b-415f-a3de-57a65731e8a5">

2. Contact Azure Support who will help implement prepend on the inbound direction of AVS leaf to achieve path preference. This will not be visible to customer and every time any change needs to be done on prepends you need to contact support.

## Scenario 2: Influencing path preference with two circuits
In this example one ExpressRoute circuit is in Dallas and another one in Chicago. We want to use Dallas as the primary path and Chicago as the Secondary/failover path.

![image](https://github.com/shrnair/AS-path-Prepends-with-Azure-VMware-Solution/assets/47249875/cb1879ab-b08a-4f90-bf76-60a76a267b22)

All path a, b, c and d will advertise on-prem prefix 10.61.0.0/24.
Path a and b will advertise 10.61.0.0/24 without any additional prepends. I will only put sample outputs from MSEE1. Both MSEE1 and MSEE2 routes would be identical.

What MSEE1 sees?

<img width="353" alt="image" src="https://github.com/shrnair/AS-path-Prepends-with-Azure-VMware-Solution/assets/47249875/51e70385-4944-4015-931a-26b9c312c556">

Path c and d will advertise 10.61.0.0/24 with prepends.

What MSEE3 sees?

<img width="466" alt="image" src="https://github.com/shrnair/AS-path-Prepends-with-Azure-VMware-Solution/assets/47249875/bdf33422-5797-4d9d-8e04-54aa0c280e52">

What DMSEE1 sees?

The BGP routing table has both routes from path a and c. But Path a is preferred as it is shorter 

<img width="461" alt="image" src="https://github.com/shrnair/AS-path-Prepends-with-Azure-VMware-Solution/assets/47249875/c3bd4e7c-8aa0-4bdb-b1e5-af9fdf1ece5a">

What DMSEE2 sees?

The BGP routing table has both routes from path b and d. But Path b is preferred as it is shorter

<img width="439" alt="image" src="https://github.com/shrnair/AS-path-Prepends-with-Azure-VMware-Solution/assets/47249875/d3c5e75a-8bfb-422c-987c-16092a6b2df3">

How is the route advertised to AVS Edge?

<img width="452" alt="image" src="https://github.com/shrnair/AS-path-Prepends-with-Azure-VMware-Solution/assets/47249875/08793faf-ed1d-4166-8e80-7cfa6f5a55f3">

The ASNs are still stripped of by the DMSEE. But since both DMSEE have identical routes and both identify Dallas(a and b) as shorter path. Hence the traffic from AVS to On-Prem will always route through Dallas circuit. Unless the routes are lost or the circuit is down.

## Scenario 3:Need internet breakout from AVS to be routed through an Azure hub. Need to have a backup Internet path through another hub.

There are two Hub with two NVAs each(I used CSR in my setup) advertising 0/0 using ARS and have next hop as the ILB IP1 and ILB IP2.

<img width="647" alt="image" src="https://github.com/shrnair/AS-path-Prepends-with-Azure-VMware-Solution/assets/47249875/d0a767cd-64d8-48eb-b2c5-bf1b5840fd7a">

What DMSEE1 sees?

It receives 0/0 from both Vnet-prod and Vnet-dev.
Vnet-prod is the preferred path due to shorter As-path length

<img width="497" alt="image" src="https://github.com/shrnair/AS-path-Prepends-with-Azure-VMware-Solution/assets/47249875/160a49be-be81-4eb7-a8cf-562d79371f5f">

What DMSEE2 sees?

It is identical to DMSEE1 routes

<img width="528" alt="image" src="https://github.com/shrnair/AS-path-Prepends-with-Azure-VMware-Solution/assets/47249875/654e5cdd-494e-497c-acb7-d37d5280bcf9">

What DMSEE advertises to AVS edge?

<img width="439" alt="image" src="https://github.com/shrnair/AS-path-Prepends-with-Azure-VMware-Solution/assets/47249875/c2b4c25c-712b-4f65-8fc7-7880ab962d2f">

The ASNs are still stripped of by the DMSEE. But since both DMSEE have identical routes and both identify path a as shorter path due to As-path length. Hence the traffic from AVS to Internet will always route through Vnet-Prod. Unless the routes are lost from Vnet-Prod.








