---
layout: post
title: use a Private Endpoint for your ADX cluster
subtitle: use private endpoints to secure your cluster
thumbnail-img: /assets/img/20210901/thumb.png
cover-img: /assets/img/20210901/banner.png
share-img: /assets/img/20210901/banner.png
tags: [Azure, Time Series Insights, TSI, REST API]
comments: true
---


FRAGEN:
- die zwei Storagfe Account sind quasi ein cache?
- Screenshot network isolation nochmal mit DNS private zone machen



Keeping your data secure in the cloud should be one of the most prioritized task of your IT department. Having an ADX cluster in the cloud which processes, stores and analyzes your business critical information should be securely operated. Therefore Microsoft currently offers the capability to create an ADX cluster within a subnet in your Azure virtual network. This enables you to access your cluster privately from your VNet or on-premises network and creates capabilities to access resources such as Event Hub and storage inside your network. Additionally you can restrict inbound and outbound traffic!

To get all of the available security features as described above, you would need to setup an Azure Private Link Service as well as an Azure Private Endpoint and additionally configure and change existing DNS configuration. All of this requries a bit of work and in some special cases the overall process is a bit inconvenient!

There is the private preview for a native integration of private endpoints currently ongoing which i was lucky enough to join. This blogpost should provide your with an common understanding how you can use this feature make your ADX clusters even more secure!

The final integration of the private endpoint in ADX will probably look like what we know from other services like storage, cosmos db, ... 
{: .alert .alert-warning}

This private preview is going to be configured differently from what I suspect you will see later in the portal or via CLI. The steps I have to do manually will be all integrated in an dedicated tab in the service! Only the end result will look similar. 

OK, so lets get started ....

# creation of an ADX cluster

before creating a private endpoint for your ADX cluster, you obviously need the ADX cluster itself. Later in the post i will cover a few more tricks on how to make your ADX even more secure, but for the purpose of the demo, I will just use a standard ADX cluster deployment even via the portal!

For this purpose you do not need to configure network isolation.

![](/assets/img/20210916/adxcreation.png) 

Once your cluster got created, you should see your resource in the portal!

![](/assets/img/20210916/adxcreation2.png)

# secure ADX and enable a private endpoint

Like I said in the beginning, your implementation of ADX private endpoint will look differently, the end result will be the same!

## creating a private endpoint

creating a private endpoint for ADX will create a dedicated network interface for this auzre service in your VNet. This will enable clients / consumers to securely connect to this service. Network traffic between azure services (e.g eventhub and ADX) traverses over the vnet and the private link on the Microsoft backbone network, eliminating exposure from the public internet.

![](/assets/img/20210916/peadx.png)

The resource type *Microsoft.Kusto/clusters* is the private preview feature which is not available for you as of now!

![](/assets/img/20210916/peadx2.png)

After all resources got provision, you will see those in the portal. In the next screenshot we compare the newly created resources against the resources created when doing network isolation.

![](/assets/img/20210916/adxpeenabled.png)

Compared to all the resources we get for network isolation, it looks a lot cleaner.

![](/assets/img/20210916/adxcomparenetworkisolation.png)

 # taking a more closer look at those provisioned resources

Lets take a closer look at those provisioned resources to understand how those additional resources interconnect to each other and where you can use levers to strengthen the clusters security!

## network interface card

Like with any other private endpoint you will get a dedicated network interface card, bound to the private endpoint of your selected service type. This will ensure that your service will receive a private IP Address of the network you have selected. Instead of having just a public endpoint, your service now has already a foot in your network. But this is for sure not the only thing to get all working.

## private DNS zone

it is quite important to correctly configure your DNS settings to resolve the private IP address to the FQDN of the connection string of your service.

Before using the private preview, in case we wanted to use private links with ADX, you needed to configure the DNS settings on your own. With the private preview and hopefully later in the product, you do not need to take care about this at all. (In case you stick to the standard configuration) 
<br>For the purpose of the demo I just wanted to explain the single resources to better understand the process!

![](/assets/img/20210916/dnszonesrg.png)

As you can see, Azure created four private DNS zones for us. The names already reveal what other services are used in the backend. Blob, Table and Queue obviously belong to an azure storage account service. The reason why those are listed there is being discussed later in the blogpost. The other private DNS zone belongs to the actual ADX service itself.

The reason why the private endpoint created those private DNS zones is that you do not change your consumers connection strings to automatically resolve the private IP address of the service.<br>
First Azure will create a CNAME record in the public DNS zone to redirect the resolution to the private domain name. You can see to which private DNS zones azure will configure the CNAME record.
[Azure Services DNS zone configuration](https://docs.microsoft.com/en-us/azure/private-link/private-endpoint-dns#azure-services-dns-zone-configuration)
<br>
<br>Now your request e.g. adxdemoblog.westeurope.kusto.windows.net resolves to adxdemoblog.privatelink.westeurope.kusto.windows.net

This configuration is going to happen automatically, but now, your provisioned private DNS zones become functional. 

Your consumer / application now tries to query for the private endpoint IP address in azure provided DNS service. Azure DNS will be responsible for DNS resolution of the private DNS zone. You can take a closer look to the private DNS zone for the kusto service in that case.

![](/assets/img/20210916/pdnszadx.png)


This configuration shows, that there is an A-Type DNS record in the private DNS zone *privatelink.westeurope.kusto.windows.net*. If your client is in the same network than the private dns zones is linked to, it can resolve the A record to its internal, private IP address.

## private endpoint

The private endpoint resource type just does the linking between the NIC address and the private DNS configuration that a private connection can be established!

# Making your ADX more secure







