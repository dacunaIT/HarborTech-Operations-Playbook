
# Week 9: VPC Networking and Troubleshooting

## HarborTech Ticket Summary

Ticket TKT-2026-0009 reports that an internet client cannot reach an EC2 web application using TCP port 80. The EC2 instance is running, and the web service is listening, but external connectivity has not been established.

The investigation focuses on the VPC configuration, including IP addressing, subnet routing, internet gateway connectivity, security groups, and network ACLs.

The evidence suggests that the primary issue is a missing default route to the internet gateway rather than a problem with the application itself.

## Client Impact

The client is experiencing an application availability issue because external users cannot reach the web server.

Even when an EC2 instance is functioning correctly, incorrect routing or subnet configuration can prevent users from accessing its services.

This can interrupt business operations, reduce application availability, and prevent customers from accessing important resources.

## AWS Services Involved

The following AWS networking services and concepts are relevant to this investigation:

- **Amazon VPC:** Provides an isolated virtual network for AWS resources.
- **Subnets:** Divide a VPC into smaller network segments.
- **Route Tables:** Determine where network traffic is directed.
- **Internet Gateway:** Provides a path between a VPC and the public internet.
- **NAT Gateway:** Allows private instances to initiate outbound internet connections without directly accepting unsolicited inbound connections.
- **Security Groups:** Stateful firewalls controlling traffic to and from EC2 instances.
- **Network ACLs:** Stateless security controls operating at the subnet level.
- **CIDR Blocks:** Define IP address ranges within networks.
- **Public Subnets:** Subnets with an active default route to an internet gateway.
- **Private Subnets:** Subnets without a direct default route to an internet gateway.
- **VPC Flow Logs:** Record network traffic metadata for troubleshooting and monitoring.

## Virtualization Connection

Amazon VPC uses software-defined networking to provide network functionality without requiring users to manage physical routers, switches, or firewalls.

AWS allows administrators to configure virtual subnets, routing tables, gateways, and security policies through the AWS Management Console or CLI.

Although these resources are virtualized, they still follow traditional networking principles.

Traffic requires valid addressing, a working route, appropriate security permissions, and a return path.

This investigation demonstrates that a virtual server can be running correctly while its network configuration prevents external communication.

## Evidence Reviewed

The following information was provided in the simulated HarborTech evidence packet. This was training evidence rather than information collected directly from my AWS Learner Lab.

### Required Traffic Flow

- Source: Internet client
- Destination: EC2 web server
- Protocol: TCP
- Destination port: 80
- Expected service: HTTP web application

### Addressing and Subnet Evidence

- The EC2 instance is running.
- The web service is listening.
- The instance has a public IPv4 address.
- The instance belongs to a subnet within the VPC.
- The ticket describes the subnet as public.

A public IPv4 address alone does not prove that the instance has a working internet route.

### Route Table Evidence

The subnet's associated route table contains a local VPC route.

However, the evidence does not show a default route directing internet traffic to an internet gateway.

Expected public subnet routing:

Destination: 0.0.0.0/0  
Target: Internet Gateway (igw-...)

The missing default route is the most important finding in the investigation.

### Internet Gateway Evidence

The VPC has an attached internet gateway.

However, an attached internet gateway does not automatically provide connectivity.

The subnet must also have a route table entry directing internet-bound traffic to that gateway.

### Security Group Evidence

The security group permits inbound TCP traffic on port 80 from the required source.

This supports the intended HTTP connection and makes the displayed security group rule less likely to be the cause of the problem.

### Network ACL Evidence

The network ACL permits the required inbound TCP port 80 traffic and outbound response traffic using ephemeral ports.

These rules support the expected request and response flow.

However, the full rule sets were not provided, so additional restrictions cannot be completely ruled out.

### Connectivity Evidence

No completed external HTTP connectivity test or VPC Flow Log results were included in the evidence packet.

Therefore, the investigation identifies a likely configuration problem but does not claim that connectivity has been successfully restored.

## Operational Analysis

The required traffic path is:

Internet Client
→ Internet Gateway
→ VPC Subnet
→ EC2 Security Group
→ EC2 Web Server

The returning traffic must also have a valid route back toward the internet client.

The instance has a public IPv4 address, and the VPC has an attached internet gateway.

However, the subnet's associated route table contains only the local VPC route.

This means the evidence does not support the ticket's assumption that the instance is located in a properly configured public subnet.

The first unsupported assumption is that a public IP address automatically makes the subnet publicly reachable.

The security group and network ACL appear to permit the required HTTP traffic, making those layers less likely to be responsible.

The most likely blocking layer is the subnet routing configuration.

Without an active default route to the internet gateway, the required public internet traffic path is incomplete.

### Public and Private Subnet Comparison

A public-facing web server generally requires a public IP address and a subnet route to an internet gateway.

A private application server that only needs to download operating-system updates should instead use a NAT gateway for outbound connectivity.

For that design, the private subnet's default route would point to a NAT gateway located in a public subnet.

The NAT gateway's public subnet would then require a default route to an internet gateway.

This provides outbound access without directly exposing the private instance to unsolicited inbound internet traffic.

## Recommendation

The recommended first step is to verify the subnet's route table association and confirm the internet gateway attachment.

If the application is intended to be publicly accessible, the subnet should have an active default route:

Destination: 0.0.0.0/0  
Target: Attached Internet Gateway

This change should only be made after confirming the intended network design and receiving the appropriate approval.

There is no evidence supporting unnecessary changes to the security group or network ACL.

After an approved correction, the next step is to test HTTP connectivity from an external client using the instance's public IPv4 address.

An example verification command is:

```bash
curl -I http://PUBLIC_IP_ADDRESS
```

A successful HTTP response would help confirm that the web application is reachable.

If the connection still fails, additional investigation should include:

- Confirming the instance's public IP address
- Reviewing the complete security group rules
- Reviewing the complete network ACL rules
- Checking VPC Flow Logs
- Using VPC Reachability Analyzer
- Confirming that the web service is listening on the expected interface and port

No production changes or live connectivity tests were performed as part of the provided simulated evidence.

## Escalation Notes

Any production route table modification requires appropriate authorization before implementation.

The proposed change should be reviewed to ensure that the subnet is intended to provide public internet access.

If the application is supposed to remain private, the issue may require an architecture review rather than adding a direct internet gateway route.

Changes to production security groups, network ACLs, internet gateways, or NAT gateways should also follow the organization's change management process.

The current evidence supports recommending a routing correction, but not making unapproved production changes.

## Lessons Learned

This investigation demonstrated that a running EC2 instance does not guarantee network connectivity.

A public IP address and an attached internet gateway are not enough to establish public access without the correct route table configuration.

I also learned that security groups, network ACLs, and route tables perform different functions.

Security groups and network ACLs determine whether traffic is allowed, while route tables determine where traffic is sent.

Troubleshooting should follow the complete traffic path rather than immediately changing familiar security settings.

The most important lesson was to identify the first unsupported assumption, use available evidence to narrow the issue, and recommend the smallest necessary correction.

## Professional Vocabulary

- **VPC:** A logically isolated virtual network within AWS.
- **Subnet:** A smaller IP address range within a VPC where resources can be deployed.
- **Route Table:** A collection of rules determining where network traffic is forwarded.
- **Internet Gateway:** A VPC component that enables communication with the public internet.
- **NAT Gateway:** A managed service that allows private resources to initiate outbound connections.
- **Security Group:** A stateful firewall controlling allowed traffic for AWS resources.
- **Network ACL:** A stateless firewall that controls traffic entering and leaving a subnet.
- **CIDR:** A notation used to define network IP address ranges.
- **Public Subnet:** A subnet whose associated route table has an active route to an internet gateway.
- **Private Subnet:** A subnet without a direct default route to an internet gateway.
- **Default Route:** A route, commonly 0.0.0.0/0, used when no more specific route matches.
- **Ephemeral Ports:** Temporary ports commonly used for client connections and response traffic.
- **VPC Flow Logs:** Records of network traffic metadata used for monitoring and troubleshooting.
- **Reachability Analyzer:** An AWS tool that analyzes network paths between supported resources.
- **Software-Defined Networking:** Networking controlled through software configurations rather than direct management of physical networking equipment.
