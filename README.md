# Private AKS Cluster with Restricted Egress and its accessibility across VNets 

Provisioning a Kubernetes cluster in any cloud platform is a trivial process for a user. But when we setup this in an enterprise where multiple Kubernetes clusters are run and there is a need to secure the cluster, it become a challenge. I also faced such a use case and its challenges. In this due course, had few learnings which I would like to share. 

Consider an example of Contoso organization which uses a Hub and Spoke network topology.  The Hub VNet is created in a separate subscription and the Spoke VNets are in separate subscriptions – all belonging to the same Tenant. In the Hub VNet they have a firewall, and all spokes are connected to the Hub via VNET Peering. They have multiple private AKS clusters in each spoke and they want to deploy a centralized monitoring cluster which has Spinnaker and Prometheus. This monitoring AKS cluster is responsible for connecting to other AKS clusters residing in other spokes in different subscriptions. The key challenges which we came across in this requirement are:

1. Restricting egress for AKS private cluster via Firewall
2. Route Spoke-Spoke traffic via Hub 
3. Connecting private AKS cluster across VNetsresiding in different subscriptions. 

We setup an environment to test the above use case. The setup looks as below:

1. A Hub and Spoke network topology with 1 Hub and 3 Spokes VNETs (this can be in different subscriptions).
2. Spoke 1, 2 and 3 are peered with Hub VNET.
3. There is no direct peering between spokes.
4. Spoke 1 and Spoke 3 have one private AKS cluster each. Spoke 2 has one private AKS cluster (monitoring cluster having spinnaker).
5. Hub will have an Azure Firewall deployed
6. One VM in each of the Spoke VNET to test the connectivity.
7. A VNet Gateway (VPN) in the Hub

> For the sake of simplicity, we will scope our aim in trying to establish connectivity between a VM provisioned in Spoke 2 and the private AKS cluster provisioned in Spoke 1.

## Network Architecture

![Network Architecture](/images/Hub_Spoke_Figure1.png)

## Set Up

The series of steps involves

1. Create the Spoke VNets and peer them to the central HUB VNet. When you peer, you need to ensure the peered VNet has forward traffic enabled as shown in figure below.

   ![VNET Peering Settings](/images/VNETPeer.png)

2. Provision an Azure Firewall in the Hub VNet. Alternatively, you can use any other NVA appliance

3. Create a route table and attach it to the Spoke AKS subnets as shown in Figure 1 above. 0.0.0.0/0 indicates all internet bound requests and these requests should be routed to Firewall Private IP address

   ![Route Table](/images/RouteTable.png)

4. Define the below network firewall rules to allow outbound requests to Azure IPs. If you need fine grained CIDRs please refer here.

   ![Firewall Network Policies](/images/FirewallNWPolicies.png)

   Application Rule. Note: To create a FQDN based rule on firewall you need to enable DNS proxy on Firewall.

   ![Firewall App Policies](/images/FirewallAppPolicies.png)

   Under firewall policy -> DNS enable the proxy

   ![Firewall DNS Settings](/images/FirewallDNSSettings.png)

5. Create a AKS private cluster with Managed Identity in SPOKE1. Since we are ensuring the egress traffic flows via firewall, we need create the cluster with outbound-type as userDefinedRouting. This will ensure no public LB is created. The default value is LoadBalancer, which create a public LB and routes all traffic via that LB.

   ![AKS Create Command](/images/AKSClusterCreationCommand.png)

    ` az aks create -g aksoutboundtest -n privateaksspoke2 -l eastus2 --node-count 1 --generate-ssh-keys --network-plugin Azure --outbound-type userDefinedRouting --service-cidr  10.41.0.0/16 --dns-service-ip 10.41.0.10 --docker-bridge-address 172.17.0.1/16 --vnet-subnet-id /subscriptions/14xxxxx-xxxx-xxxx-xxxx-xdxxdxxxxd/resourceGroups/aksoutboundtest/providers/Microsoft.Network/virtualNetworks/SPOKE1/subnets/SPOKE1_AKS --enable-private-cluster --enable-managed-identity `

6. You need to give “Network Contributor” access to the newly created managed identity on SPOKE1 VNET where AKS is now deployed. Use the command to see the details on the newly created cluster. The principal Id shown below is your clientId.

   `$ az aks show --name privateaksspoke1 -g aksoutboundtest `

7. Connect to the AKS cluster and try to list the resources.

   ` $ az aks get-credentials --resource-group aksoutboundtest --name privateaksspoke1 `

   ![Az AKS Get-Credentials Command](/images/AKSKubeCtlShow1.png)

8. Now let’s see if we can connect it from in SPOKE 2 VM. Spoke 2 VNET is connected via HUB, so all the traffic from SPOKE2 to SPOKE1 can traverse via HUB.

   ![Az AKS Get-Credentials Command Error](/images/AKSKubeCtlShow2.png)

   Oh!! This is not resolving the master DNS name, now let’s look how we can resolve this from another VNET even if its running in different subscription. Note the private   cluster API Address from cluster over page as shown below.

   ![API Server Address](/images/APIServerAddress.png)

   Find the private DNS zone matching the API address (exclude first hostname). In the virtual network links by default, it is linked to the VNET where the cluster is created as shown below.

   ![Private DNS Show](/images/PrivateZoneLinkVNET.png)

   Now let’s link the new VNET. Click +Add. As shown in the below screenshot, we can link any VNET from any subscription in the same tenant.

   ![Private DNS Add](/images/PrivateZoneLinkVNETAdd.png)

   As show below its added

   ![Private DNS Complete](/images/PrivateZoneLinkVNETComplete.png)

   Now let’s try again.

   ![Kubectl Command Error](/images/AKSKubeCtlShow3.png)

   Since all the traffic goes via firewall, we need to allow the API server IP address in the firewall. In the above private zone overview, you can find the A name entry and the private IP address of API server. Allowed it in firewall network policy. 

   ![Firewall Master Whitelisting](/images/FirewallNWPolicies2.png)

   Now let’s try again and it should work.

   ![Kubectl Command Successful](/images/AKSKubeCtlShow4.png)

> Note: Ensure you have the route table attached to all the SPOKE VNETs

   Now we see that the test VM in Spoke 2 is able to connect to a private AKS cluster running in Spoke 1. The connection between the VM in Spoke 2 and the AKS cluster is happening via the Azure Firewall.

   So, we are able to overcome all the challenges mentioned earlier:

    1. Restricting egress for AKS private cluster via Firewall
    2. Route Spoke-Spoke traffic via Hub
    3. Connecting private AKS cluster across VNets residing in different subscriptions

Special thanks to @saurabhvartak1982  - Cloud Solution Architect @ Microsoft

We worked together on this use case and co-authored this article

