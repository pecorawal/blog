# Protect your ROSA cluster with Openshift Dedicated and Transit Gw 


A systems administrator must make numerous choices on the infrastructure environment that underlies the applications supported by a company. Undoubtedly, one of the most challenging issues is having to choose between staying in a traditional environment and moving to the cloud, regardless of the cloud provider.
I can now assist you in making the challenging decision to move your workloads to the cloud while maintaining a minimum level of security.
The choices for using Openshift Dedicated as the platform for your apps will be covered in this post. The cluster, its management interface, and any applications that are exposed must first be made public or private. It is crucial to emphasize that making anything public or private has no relevance on its integrity.



## Options for connecting to the private cluster


When you make your cluster private, no one will be able to access it; to achieve this, connect the cluster's VPC to an AWS VPC under your account for complete traffic control.

Amazon offers three different kinds of objects for the connection:
 
   * Transit Gateway
   * Vpc Peering
   * Direct Connect.

Networks can peer with one another through VPC similar to being connected by a Lan-to-Lan link.
The alternative approach is Direct Connect, which uses a secure connection that functions like a VPN between a managed cluster and a customer environment.
Finally, the Transit Gateway is the main topic of our article.
An AWS network object called the Transit Gateway enables routing between the cluster's VPC Network and a number of other VPCs in the cloud.
So, let's start it!


# Creating objects

Now it's time to create the necessary objects for our private cluster.
The following steps will be required to link your private cluster to an AWS environment using Transit Gw.

1. Create the Private VPC for the Cluster
   1.1 Create Private Subnet for the Egress VPC
2. Create Private ROSA cluster
3. Create the public VPC that will connect the cluster to the internet, we will call it Egress VPC
4. Allow DNS between vpcs
5. Create subnets and network stuff.
 
## Create private VPC - CLUSTER VPC

In case to be a new subscription, do the next step, otherwise, skip it:

```
$ aws iam create-service-linked-role --aws-service-name "elasticloadbalancing.amazonaws.com"
```

```
export VERSION#4.12.0 \
       ROSA_CLUSTER_NAME#myprivate-rosa \
       AWS_DEFAULT_REGION#us-east-1

VPC_ID_1#`aws ec2 create-vpc --cidr-block 10.0.0.0/16 | jq -r .Vpc.VpcId`

aws ec2 create-tags --resources $VPC_ID_1 --tags Key#Name,Value#rosa_intranet_vpc
```

!(.images/1-create-private-vpc-clusterside.png))["VPC Cluster Side"]



## Create a private subnet based on Private VPC - THIS IS THE CLUSTER WILL RESIDE ON IT

```
ROSA_PRIVATE_SUBNET#`aws ec2 create-subnet --vpc-id $VPC_ID_1 --cidr-block 10.0.0.0/17 | jq -r .Subnet.SubnetId`
aws ec2 create-tags --resources $ROSA_PRIVATE_SUBNET --tags Key#Name,Value#intranet-pvt
echo $ROSA_PRIVATE_SUBNET
```
!(.images/3-private-subnet-vpc.png)[Cluster Private Subnet]


## Create ROSA cluster - Single Zone

Deploy your ROSA private cluster using Red Cloud Console named Openshift Cluster Manager at https://console.redhat.com/openshift or via cli sample below.

```
rosa create cluster --private-link --cluster-name#$ROSA_CLUSTER_NAME [--machine-cidr#10.0.0.0/16] --subnet-ids#$ROSA_PRIVATE_SUBNET
```

IMPORTANT: ROSA cli sample requires the prerequisites creation described on Red Hat Official Documentation. 


## Create Egress VPC
 
```
VPC_ID_2#`aws ec2 create-vpc --cidr-block 172.100.0.0/20 | jq -r .Vpc.VpcId`
echo $VPC_ID_2
aws ec2 create-tags --resources $VPC_ID_2 --tags Key#Name,Value#egress_vpc
```

!(.images/2-create-vpc-customerside.png)["VPC Customer AWS Subscription"]


## Setup DNS

```
aws ec2 modify-vpc-attribute --vpc-id $VPC_ID_1 --enable-dns-hostnames
aws ec2 modify-vpc-attribute --vpc-id $VPC_ID_2 --enable-dns-hostnames
```


## Private Subnet for Egress VPC

TIP: In our example we are creating Egress VPC and their subnets, therefore you can skip this tasks if you already have this underlying infrastructure.

```
EGRESS_PRIVATE_SUBNET#`aws ec2 create-subnet --vpc-id $VPC_ID_2 --cidr-block 172.100.0.0/17 | jq -r .Subnet.SubnetId`
aws ec2 create-tags --resources $EGRESS_PRIVATE_SUBNET --tags Key#Name,Value#egress-pvt
echo $EGRESS_PRIVATE_SUBNET
```
!(.images/4-private-subnet-Egressvpc.png)[Egress VPC - Private Subnet]


## Public Subnet for Egress VPC

```
EGRESS_PUBLIC_SUBNET#`aws ec2 create-subnet --vpc-id $VPC_ID_2 --cidr-block 172.100.128.0/17 | jq -r .Subnet.SubnetId`
aws ec2 create-tags --resources $EGRESS_PUBLIC_SUBNET --tags Key#Name,Value#egress-public
echo $EGRESS_PUBLIC_SUBNET
```
!(.images/5-public-subnet-Egressvpc.png)[Egress VPC - Public Subnet]



## Create Internet Gateway - only for Egress VPC

Internet Gateway is the AWS object responsible to create a way to VPC access internet. In order to ensure the security of your clusters, it is recommended to limit internet output to the VPC environment, where network traffic from your other environments is regulated. We are creating the networks from each cluster's perspective and the client-controlled environment's perspective as this is a proof of concept.

```
INTERNETGW#`aws ec2 create-internet-gateway | jq -r .InternetGateway.InternetGatewayId`
echo $INTERNETGW
aws ec2 create-tags --resources $INTERNETGW --tags Key#Name,Value#igw-osd-neon
```





## Attach Internet Gateway to Egress VPC

After created you should attach the Internet Gateway to the Egress VPC.

```
aws ec2 attach-internet-gateway --vpc-id $VPC_ID_2 --internet-gateway-id $INTERNETGW
```

!(.images/6-igw.png)[Internet Gateway]

## Create Nat Gateway for Public Egress Subnet

Create a public egress subnet to allow egress traffic thru the egress public subnet only. Associate an Elastic IP to guarantee

```
ELASTICIP#`aws ec2 allocate-address --domain vpc | jq -r .AllocationId`
echo $ELASTICIP
NAT_GATEWAY#`aws ec2 create-nat-gateway --subnet-id $EGRESS_PUBLIC_SUBNET --allocation-id $ELASTICIP | jq -r .NatGateway.NatGatewayId`
echo $NAT_GATEWAY
aws ec2 create-tags --resources $ELASTICIP --resources $NAT_GATEWAY --tags Key#Name,Value#egress_nat_public
```

!(.images/7-natgw.png)[Nat Gateway]

## Create AWS Transit Gateway

Create Transit GW to attach two VPCs.

```
TRANSITGW#`aws ec2 create-transit-gateway | jq -r .TransitGateway.TransitGatewayId` 
echo $TRANSITGW
aws ec2 create-tags --resources $TRANSITGW --tags Key#Name,Value#osd-neon-transit-gateway
```

!(.images/7-natgw.png)[Nat Gateway]


### Attachment to private subnet from private CLUSTER VPC to Transit GW

Transit GW starts on pending state, wait a couple o minutes until available state. After that, create a Transit GW VPC attachment on the private VPC with private subnet.

```
TRANSITGW_A_RPV#`aws ec2 create-transit-gateway-vpc-attachment --transit-gateway-id $TRANSITGW --vpc-id $VPC_ID_1 --subnet-ids $ROSA_PRIVATE_SUBNET | jq -r .TransitGatewayVpcAttachment.TransitGatewayAttachmentId`
echo $TRANSITGW_A_RPV

aws ec2 create-tags --resources $TRANSITGW_A_RPV --tags Key#Name,Value#transit-gw-intranet-attachment
```
!(.images/9-attachment-tgw-cluster.png)[Attachment for Transit GW and Cluster VPC]


### Attachment to private subnet from Egress VPC to Transit GW

Create a transit gateway

```
TRANSITGW_A_EPV#`aws ec2 create-transit-gateway-vpc-attachment --transit-gateway-id $TRANSITGW --vpc-id $VPC_ID_2 --subnet-ids $EGRESS_PRIVATE_SUBNET | jq -r .TransitGatewayVpcAttachment.TransitGatewayAttachmentId`
echo $TRANSITGW_A_EPV
aws ec2 create-tags --resources $TRANSITGW_A_EPV --tags Key#Name,Value#transit-gw-egress-attachment
```

!(.images/10-attachment-tgw-egress.png)[Attachment for Transit GW and Egress VPC]


### Create Egress gateway route


Discover the default transit gateway's route table ID:

```
TRANSITGW_D_RT#`aws ec2 describe-transit-gateways --transit-gateway-id $TRANSITGW | jq -r '.TransitGateways | .[] | .Options.AssociationDefaultRouteTableId'`
echo $TRANSITGW_D_RT

aws ec2 create-tags --resources $TRANSITGW_D_RT --tags Key#Name,Value#transit-gw-rt
```

!(.images/11-tgw-defaultroute.png)[Transit Gw Default Route]


### Create static route for internet traffic to go to the egress VPC

```
aws ec2 create-transit-gateway-route --destination-cidr-block 0.0.0.0/0 --transit-gateway-route-table-id $TRANSITGW_D_RT --transit-gateway-attachment-id $TRANSITGW_A_EPV

```
!(.images/12-egresspublic-igw-table.png)[Static Route to Egress Public Subnet for Internet Traffic]


#### Discover the main route table to private VPC

```
ROSA_VPC_MAIN_RT#`aws ec2 describe-route-tables --filters 'Name#vpc-id,Values#'$VPC_ID_1'' --query 'RouteTables[].Associations[].RouteTableId' | jq .[] | tr -d '"'`
echo $ROSA_VPC_MAIN_RT

aws ec2 create-tags --resources $ROSA_VPC_MAIN_RT --tags Key#Name,Value#rosa_main_rt 
```


#### Discover the main route table to egress VPC

```
EGRESS_VPC_MAIN_RT#`aws ec2 describe-route-tables --filters 'Name#vpc-id,Values#'$VPC_ID_2'' --query 'RouteTables[].Associations[].RouteTableId' | jq .[] | tr -d '"'`
echo $EGRESS_VPC_MAIN_RT
```

### Create a private route table in egress VPC

```
EGRESS_PRI_RT#`aws ec2 create-route-table --vpc-id $VPC_ID_2 | jq -r .RouteTable.RouteTableId`
echo $EGRESS_PRI_RT

aws ec2 associate-route-table --route-table-id $EGRESS_PRI_RT --subnet-id $EGRESS_PRIVATE_SUBNET
```



### Create NAT gateway route

Create a route in the egress private route table for all addresses to the NAT gateway

```
aws ec2 create-route --route-table-id $EGRESS_PRI_RT --destination-cidr-block 0.0.0.0/0 --gateway-id $NAT_GATEWAY 
```

!(.images/13-egresspublic-natgw-table.png)[Route table to egress private Nat Gateway]

Create a route in the egress VPC's main route table for all addresses going to the internet gateway

```
aws ec2 create-route --route-table-id $EGRESS_VPC_MAIN_RT --destination-cidr-block 0.0.0.0/0 --gateway-id $INTERNETGW 
```


Create a route in the egress VPC's main route table to direct addresses in the OpenShift Service on AWS private VPC to the transit gateway


```
aws ec2 create-route --route-table-id $EGRESS_VPC_MAIN_RT --destination-cidr-block 172.100.0.0/20 --gateway-id $TRANSITGW 
```

Create a route in the OpenShift Service on AWS private route table to direct all of its addresses to the transit gateway

```
aws ec2 create-route --route-table-id $ROSA_VPC_MAIN_RT --destination-cidr-block 0.0.0.0/0 --gateway-id $TRANSITGW 
```


## Conclusion

After create all objects related to Transit Gateway and associate the static routes and route tables to the desired networks you can control the traffic between a private cluster to subnets inside customer's VPCs. This kind of approach can route which networks (more than once, as shown) are allowed to access the infrastructures even in the cloud or on-premises.

In our example, I showed how to create anything from VPCs and private clusters to route tables for various networks. You may already have Internet Gateway and Nat Gateway established in your Amazon environment, in which case you can only deploy Transit Gateway and their attachments.

Remember that at the conclusion you will have a diagram that resembles the figure below that you will use as a reference for your documentation as well.


!(.images/full-diagram.png)[Openshift Dedicated Private Cluster with Transit Gateway]



## References

https://docs.openshift.com/rosa/rosa_install_access_delete_clusters/rosa-aws-privatelink-creating-cluster.html
https://aws.amazon.com/blogs/containers/red-hat-openshift-service-on-aws-private-clusters-with-aws-privatelink/
https://aws.amazon.com/blogs/containers/red-hat-openshift-service-on-aws-private-clusters-with-aws-privatelink/
https://access.redhat.com/documentation/en-us/openshift_dedicated/4/html-single/installing_accessing_and_deleting_openshift_dedicated_clusters/index
