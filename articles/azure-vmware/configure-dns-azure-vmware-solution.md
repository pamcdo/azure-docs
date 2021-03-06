---
title: Configure DNS forwarder for Azure VMware Solution
description: Learn how to configure DNS forwarder for Azure VMware Solution using the Azure portal. 
ms.topic: how-to
ms.custom: contperf-fy22q1
ms.date: 07/15/2021

#Customer intent: As an Azure service administrator, I want to <define conditional forwarding rules for a desired domain name to a desired set of private DNS servers via the NSX-T DNS Service.>  

---

# Configure a DNS forwarder in the Azure portal

>[!IMPORTANT]
>For Azure VMware Solution private clouds created on or after July 1, 2021, you now have the ability to configure private DNS resolution. For private clouds created before July 1, 2021, that need private DNS resolution, open a [support request](https://rc.portal.azure.com/#create/Microsoft.Support) and request Private DNS configuration. 

By default, Azure VMware Solution management components such as vCenter can only resolve name records available through Public DNS. However, certain hybrid use cases require Azure VMware Solution management components to resolve name records from privately hosted DNS to properly function, including customer-managed systems such as vCenter and Active Directory.

Private DNS for Azure VMware Solution management components lets you define conditional forwarding rules for the desired domain name to a selected set of private DNS servers through the NSX-T DNS Service.

This capability uses the DNS Forwarder Service in NSX-T. A DNS service and default DNS zone are provided as part of your private cloud. To enable Azure VMware Solution management components to resolve records from your private DNS systems, you must define an FQDN zone and apply it to the NSX-T DNS Service. The DNS Service conditionally forwards DNS queries for each zone based on the external DNS servers defined in that zone.

>[!NOTE]
>The DNS Service is associated with up to five FQDN zones. Each FQDN zone is associated with up to three DNS servers.


## Architecture

:::image type="content" source="media/networking/dns-forwarder-diagram.png" alt-text="Diagram showing that the NSX-T DNS Service can forward DNS queries to DNS systems hosted in Azure and on-premises environments." border="false":::


## Configure DNS forwarder

1. In your Azure VMware Solution private cloud, under **Workload Networking**, select **DNS** > **DNS zones**. Then select **Add**.

   >[!NOTE]
   >The default DNS zone is created for you during the private cloud creation.

   :::image type="content" source="media/networking/configure-dns-forwarder-1.png" alt-text="Screenshot showing how to add DNS zones to an Azure VMware Solution private cloud.":::

1. Select **FQDN zone** and provide a name, the FQDN zone, and up to three DNS server IP addresses in the format of **10.0.0.53**. Then select **OK**.

   It takes several minutes to complete, and you can follow the progress from **Notifications**.

   :::image type="content" source="media/networking/nsxt-workload-networking-configure-fqdn-zone.png" alt-text="Screenshot showing the required information needed to add an FQDN zone.":::

   >[!IMPORTANT]
   >While NSX-T allows spaces and other non-alphanumeric characters in a DNS zone name, certain NSX resources such as a DNS Zone are mapped to an Azure resource whose names don’t permit certain characters. 
   >
   >As a result, DNS zone names that would otherwise be valid in NSX-T may need adjustment to adhere to the [Azure resource naming conventions](../azure-resource-manager/management/resource-name-rules.md#microsoftresources).

   You’ll see a message in the Notifications when the DNS zone has been created.

1. Ignore the message about a default DNS zone. A DNS zone is created for you as part of your private cloud.

1. Select the **DNS service** tab and then select **Edit**.

   >[!IMPORTANT]
   >While certain operations in your private cloud may be performed from NSX-T Manager, you must edit the DNS service from the Simplified Networking experience in the Azure portal. 

   :::image type="content" source="media/networking/configure-dns-forwarder-2.png" alt-text="Screenshot showing the DNS service tab with the Edit button selected.":::   

1. From the **FQDN zones** drop-down, select the newly created FQDN and then select **OK**.

   It takes several minutes to complete and once finished, you'll see the *Completed* message from **Notifications**.

   :::image type="content" source="media/networking/configure-dns-forwarder-3.png" alt-text="Screenshot showing the selected FQDN for the DNS service.":::

   At this point, management components in your private cloud should be able to resolve DNS entries from the FQDN zone provided to the NSX-T DNS Service. 

1. Repeat the above steps for other FQDN zones, including any applicable reverse lookup zones.


## Verify name resolution operations

After you’ve configured the DNS forwarder, you’ll have a few options available to verify name resolution operations. 

### NSX-T Manager

NSX-T Manager provides the DNS Forwarder Service statistics at the global service level and on a per-zone basis. 

1. In NSX-T Manager, select **Networking** > **DNS**, and then expand your DNS Forwarder Service.

   :::image type="content" source="media/networking/nsxt-manager-dns-services.png" alt-text="Screenshot showing the DNS Services tab in NSX-T Manager.":::

1. Select **View Statistics** and then from the **Zone Statistics** drop-down, select your FQDN Zone.

   The top half shows the statistics for the entire service, and the bottom half shows the statistics for your specified zone. In this example, you can see the forwarded queries to the DNS services specified during the configuration of the FQDN zone.

   :::image type="content" source="media/networking/nsxt-manager-dns-services-statistics.png" alt-text="Screenshot showing the DNS Forwarder statistics.":::


### PowerCLI

The NSX-T Policy API lets you run nslookup commands from the NSX-T DNS Forwarder Service. The required cmdlets are part of the **VMware.VimAutomation.Nsxt** module in PowerCLI. The following example demonstrates output from version 12.3.0 of that module.

1. Connect to your NSX-T Server. 

   >[!TIP]
   >You can obtain the IP address of your NSX-T Server from the Azure portal under **Manage** > **Identity**.
   >
   >:::image type="content" source="media/networking/configure-dns-forwarder-4.png" alt-text="Screenshot showing the NSX-T Server IP address.":::
 
   ```powershell
   Connect-NsxtServer -Server 10.103.64.3
   ```

1. Obtain a proxy to the DNS Forwarder's nslookup service.

   ```powershell
   $nslookup = Get-NsxtPolicyService -Name com.vmware.nsx_policy.infra.tier_1s.dns_forwarder.nslookup
   ```

1. Perform lookups from the DNS Forwarder Service.

   ```powershell
   $response = $nslookup.get('TNT86-T1', 'vc01.contoso.corp')
   ```

  The first parameter in the command is the ID for your private cloud’s T1 gateway, which you can obtain from the DNS service tab in the Azure portal.
