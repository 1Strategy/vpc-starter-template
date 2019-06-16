# Notes for Branch

This branch is to support deployment of VPCs across accounts via stacksets as well as including additional templates to integrate a centralized transit gateway and update the associated routes into into the existing route tables.

Templates are to be deployed in the following order:

1. network-shared-services-transit-gateway.yaml - deploy into the shared services or organizational root account

### Please note that you will have to accept the invitations for the Transit Gateway in EACH of the child accounts before proceeding
### They will be visible here: https://us-west-2.console.aws.amazon.com/ram/home?region=us-west-2

2. network-stackset-vpc.yaml - deploy into each of the child accounts via stacksets
3. network-stackset-transit-gateway.yaml - deploy into each of the child accounts via stacksets. You will need to provide as a parameter the transit gateway
identifier (tgw-abc123) from the first template.

# 1Strategy AWS VPC template

This VPC template is a complete CloudFormation template to build out a VPC network with public and private subnets in three AWS Availability Zones.

"Public" means subnets can receive traffic directly from the Internet. Traffic outbound from a public subnet usually goes through an Internet Gateway. "Private" means that a subnet cannot receive traffic directly from the Internet. Traffic outbound from a private subnet usually goes through a NAT Gateway.

Most of the subnetting ideas come from this excellent AWS blog post on [Practical VPC Design](https://medium.com/aws-activate-startup-blog/practical-vpc-design-8412e1a18dcc#.coisizjm5).

## IP Address Layout

You'll first need to determine your IP Address CIDR block for your VPC. In this example, we are using the 10.0.0.0/8 [private address space](https://en.wikipedia.org/wiki/Private_network).

We have carved the example 10.0.0.0/8 address space into four /10 address spaces. A /10 address space is assigned to a given AWS Region.
This means we could be in four different AWS Regions. In practice we will use only two Regions, US-West-2 and US-East-1. This will support up to 64 /16 VPCs in each Region.

| Location   | AWS Region | IP CIDR       | Address Range               |
|------------|------------|---------------|-----------------------------|
| Oregon     | us-west-2  | 10.0.0.0/10   | 10.0.0.1 - 10.63.255.255    |
| Virginia   | us-east-1  | 10.64.0.0/10  | 10.64.0.1 - 10.127.255.255  |
| reserved   |            | 10.128.0.0/10 | 10.128.0.0 - 10.191.255.255 |
| reserved   |            | 10.192.0.0/10 | 10.192.0.1 - 10.255.255.255 |

### Example VPC CIDR blocks

Here are the example VPC CIDR blocks we'll be using:

```text
Oregon (us-west-2):   10.0.0.0/16
Virginia (us-east-1): 10.64.0.0/16
```

See the AWS [VPC sizing docs](https://docs.aws.amazon.com/AmazonVPC/latest/UserGuide/VPC_Subnets.html#vpc-sizing-ipv4) for more info.

You are free to use any /16 to /28 CIDR block in the RFC 1918 private address range, but the VPC CIDR range you pick for this template **should not** overlap with any existing IP CIDR address ranges, either on-prem or in another AWS VPC.

---

## VPC Template Parameters

To deploy this VPC template, you'll need to know the VPC CIDR block, the three public, and three private subnet CIDR blocks. You will also choose whether the VPC will support highly available NAT Gateways, or, by default, a more cost effective single NAT Gateway.

| Parameter                 | Description                  | Example      |
|---------------------------|------------------------------|--------------|
| _VpcCidrParam_            | IPv4 CIDR block (/16 to /28) | 10.0.0.0/16  |
| _PublicAZASubnetBlock_    | AZ A public subnet block     | 10.0.32.0/20 |
| _PublicAZBSubnetBlock_    | AZ B public subnet block     |      ""      |
| _PublicAZCSubnetBlock_    | AZ C public subnet block     |      ""      |
| _PrivateAZASubnetBlock_   | AZ A private subnet block    | 10.0.64.0/19 |
| _PrivateAZBSubnetBlock_   | AZ B private subnet block    |      ""      |
| _PrivateAZCSubnetBlock_   | AZ C private subnet block    |      ""      |
| _HighlyAvailable_         | Highly Available NAT config  |     true     |


To make it easier to specify these parameters on the command line, you can use the example Parameters files included in the `parameters/` directory.

## How to Deploy

### Prerequisites

If you'd like to deploy this stack via the command line, you'll need the AWS CLI.

### Validate/Lint Stack

```shell
aws cloudformation validate-template --template-body file://network.yaml
```

### Deploy Stack

You will need to verify you have the appropriate parameters file for the AWS Region and account/environment you want to deploy to. See `./parameters/<region>/<acct>.json`. For example `parameters/us-west-2/dev.json`.

```shell
aws cloudformation create-stack --template-body file://network.yaml --stack-name main-vpc --parameters file://parameters/us-west-2/dev.json
```

### Update Stack

```shell
aws cloudformation update-stack --template-body file://network.yaml --stack-name main-vpc --parameters file://parameters/us-west-2/dev.json
```

## Template Outputs/Exports

AWS CloudFormation supports [exporting Resource names and properties](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/using-cfn-stack-exports.html). You can import these [Cross-Stack References](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/intrinsic-function-reference-importvalue.html) in other templates.

This VPC template exports the following values for use in other CloudFormaton templates. Each export is prefixed with the **Stack Name**. For example, if you name the stack "main-vpc" when you launch it, the VPC's public route table will be exported as "_main-vpc-public-rtb_"

| Export                         | Description                                          | Example         |
|--------------------------------|------------------------------------------------------|-----------------|
| _vpc-id_               | VPC Id                                               | vpc-1234abcd    |
| _public-rtb_          | Public Route table Id (shared by all public subnets) | rtb-1234abcd    |
| _public-az-a-subnet_  | AZ A public subnet Id                                | subnet-1234abcd |
| _public-az-b-subnet_  | AZ B public subnet Id                                |        ""       |
| _public-az-c-subnet_  | AZ C public subnet Id                                |        ""       |
| _private-az-a-subnet_ | AZ A private subnet Id                               | subnet-abcd1234 |
| _private-az-b-subnet_ | AZ A private subnet Id                               |        ""       |
| _private-az-c-subnet_ | AZ A private subnet Id                               |        ""       |
| _private-az-a-rtb_    | Route table for private subnets in AZ A              | rtb-abcd1234    |
| _private-az-b-rtb_    | Route table for private subnets in AZ B              |        ""       |
| _private-az-c-rtb_    | Route table for private subnets in AZ C              |        ""       |

## License

Licensed under the Apache License, Version 2.0.
