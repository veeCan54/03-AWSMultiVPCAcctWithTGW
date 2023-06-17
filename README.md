# Multi VPC Architecture
With this Hands on we explore an architecture with 3 VPCS and two different methods to connect them. <br>
This is part of the Networking in AWS Workshop Studio [here](https://catalog.workshops.aws/networking/en-US/beginner/lab1).
With Amazon VPC we are able to provision a virtual network - a logically isolated section where we can launch our AWS resources. This network can be divided into logical sub networks - subnets and we can choose to provide internet connectivity to one or all subnets as we need. An IGW - internet gateway gives the VPC internet connectivity for which routes need to be configured in the main route table. 

This is what it will look like after the 3 VPCs are set up. 
![Alt text](../03-AWSMultiVPCAcctWithTGW/images/architecture.png)

The Hands on lab instructed to create the VPCS and configure them manually using the AWS admin console. I decided to get some more practice with CloudFormation so that is what I used. [The template is here](../03-AWSMultiVPCAcctWithTGW/files/03-AWSMultiVPCAcct-Resources.yml). The security groups have been configured with rules to allow incoming ICMP traffic to EC2 instances. 

By default EC2 instances in different VPCs are not able to communicate with EC2 instances in other VPCs using their private IP addresses.

![Alt text](../03-AWSMultiVPCAcctWithTGW/images/cannotPingVPCWithoutPeering.png)

In order for different VPCs to be able to communicate with each other it is possible to setup a VPC Peering connection between them. Important charecteristics of VPC peering connections: 
1. Peering connection can be between VPCs of the same region or different regions. It is also possible to peer a VPC with a VPC belonging to another AWS account, the owner of the other VPC needs to accept the peering request. 
2. VPC peering connections do not need an Internet Gateway.
3. Peered VPC must have non overlapping IP address ranges.
4. Peering connections are not transitive. VPC A -> VPC B and VPC B -> VPC C does not mean VPC A ->  VPC C.
5. Traffic between 2 peering connections is encrypted only for inter region VPC peering, not for same region peering.
6. Inter region peering supports IPV4 & IPV6. 
7. VPC Peering connection has no cost to setup however data transfer across peering connections is charged. 

Let's request peering connections individually using the admin console and accept it on the other end by the accepting VPC. Then we modify so that route entries for the other VPCs using their respectice CIDR ranges point to the peering connection. 

![Alt text](../03-AWSMultiVPCAcctWithTGW/images/peeringConnection.png)

Even though the basic infrastructure was setup with CloudFormation I set up the VPC Peering connections manually. 
After setting up peering connections the private ip addresses between the peered VPCs are able to ping each other.

![Alt text](../03-AWSMultiVPCAcctWithTGW/images/pingWorksAfterPeering.png)

Disadvantages of VPC Peering connections:
1. It does not scale well. As the number of VPCs increases, the complexity of managing the connections also grows
2. To establish connectivity between any two VPCs they both to have to be peered, because transitive peering is not supported. 

 **Transit Gateway** is a more scalable approach. In simple terms, Transit Gateway is a highly scalable cloud router. It connects VPCs and on-premises networks through a central hub. The architecture in our case would look like this, where a TGW is used to connect 3 VPCs. 

![Alt text](../03-AWSMultiVPCAcctWithTGW/images/tgwArchitecture.png)

Routing through a Transit Gateway operates at Layer 3 where packets are sent to a specific next hop attachment, based on the destination IP address. The route table for these VPCS need to be configured to send traffic destined to the other two VPCs to the Transit gateway. So let's do that. 

Delete the VPC Peering connection first. This presents an option in the admin console to delete the Route table entries as well which makes clean up easy. 

![Alt text](../03-AWSMultiVPCAcctWithTGW/images/deletePeeringConnection.png) 

In this example, there are only 3 VPCs involved and CIDR propagates routes via internal APIs. When a VPN or Direct Connect is connected using a Transit Gateway, then Border Gateway protocol is used for propagating routes between the on-premises network and Transit Gateway. 

Create Transit Gateway with default options and create Transit Gateway attachments. Transit Gateway attachment is a connection between resources like VPC, VPN, Direct Connect and the Transit Gateway. According to best practises it is recommended to use a separate small subnet for each transit gateway VPC attachment.
So create 2 VPCs for each subnet, 1 per AZ.

![Alt text](../03-AWSMultiVPCAcctWithTGW/images/tgwAttachmentSubnets.png)

Then create the TGW Attachment. 

![Alt text](../03-AWSMultiVPCAcctWithTGW/images/tgwAttachment.png) 

Configure TGW with default options, using Amazon ASN number 64512. The significance of the number ASN number was not clear to me. From the Direct Connect FAQ, it seems like this is for using TGW for Direct Connect.
>Autonomous System numbers are used to identify networks that present a clearly defined external routing policy to the Internet. Amazon Direct Connect requires an ASN to create a public or private virtual interface. You may use a public ASN which you own, or you can pick any private ASN number between 64512 to 65535 range.

In this particular hands on  however I don't understand the significance of this number. This lab has additional follow up labs that use the same infrastructure and is probably needed there. I will leave this ASN number 64512 here for now. 

Now we need to update the route tables of the VPCS so that any traffic bound to the other two VPCs get routed to the 
Transit Gateway. To simplify the configuration we create a single 10.0.0.0/8 route for both VPCs pointing to the Transit Gateway.

![Alt text](../03-AWSMultiVPCAcctWithTGW/images/tgwAttachmentRouteTable.png) 

After this setting the pings start working again. Now the three VPCs are connected via a Transit Gateway, using a Transit Gateway attachment in separate subnets. 

![Alt text](../03-AWSMultiVPCAcctWithTGW/images/pingWorksAfterTGW.png) 

**Cleanup**
Transit Gateway is charged by the hour for every connection to a VPC so this was one of the very first things I cleaned up. All resources created manually need to be deleted. Then the Stack can be deleted. 

## Summary

**What did I learn?**
1. VPCs by themselves cannot communicate to other VPCs. We explored two ways to connect VPCs.
    1.a VPC Peering - not scalable
    1.b Transit Gateway - highly scalable service which acts as a router. With TGW we are able to connect VPCs, VPNs and Direct Connect networks with each other. TGW Attachmment is a connection between these resources and the Transit Gateway. We setup routes in VPC route tables to point traffic to external networks to the TGW. Best practise for TGW Attachment is to have these attachments in separate smaller etworks so they are configurable. 
**Mistakes?**
I wanted to automate infrastructure provisioning. I had to debug mistakes and syntax errors in my Cloud Formation Template and the lab took me longer than it would have, had I used the admin console. However I am very glad I automated whatever I could. I used up 85% of my S3 Free Tier upload for June and it is only the 15th. I will have to watch my spending closely. I have a dev and prod account, which are in free tier in my AWS Organization so I could probably use those accounts. All in all, good learning exercise.
**TODO?**
1. Next steps in automation - modify CFT by adding a Transit Gateway. However, the way I think of it (at least now) is you wouldn't need a whole lot of environments with a TGW so you might be better off doing it manually? If it is really a lot of identical environments to configure then maybe it makes sense.. Food for thought.
2. Next level labs in AWS Networking. 












