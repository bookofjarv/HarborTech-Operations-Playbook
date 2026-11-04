===# Week 9: VPC Networking and Troubleshooting

## HarborTech Ticket Summary
HarborTech Ticket TKT-2026-0009 involves a newly deployed web application running on an Amazon EC2 instance (`198.51.100.24`) within VPC `vpc-0a91`. Although the EC2 instance is running and its local service is actively listening on TCP port 80, external internet clients cannot reach the application. Investigation of the partial VPC evidence packet reveals a disconnection between the infrastructure's deployed hardware gateways and its software-defined route configuration.

## Client Impact
When incorrect routing, misconfigured subnet associations, or faulty traffic filtering are present, client-facing applications become completely unreachable despite being powered on and healthy at the compute layer. This causes service outages, disrupts business operations, damages user trust, and prevents critical administrative workflows or updates from executing successfully.

## AWS Services Involved
- **Amazon VPC:** Provides the isolated virtual network container defining the overall IP address space.
- **Subnets:** Segregate IP blocks within the VPC to house compute workloads.
- **Route Tables:** Direct packet forwarding paths containing explicit rules mapping destination CIDRs to specific gateways.
- **Internet Gateway (IGW):** Connects the VPC edge directly to the public internet.
- **NAT Gateway:** Enables secure outbound-only internet connectivity for instances in private subnets.
- **Security Groups:** Stateful instance-level firewalls controlling inbound and outbound traffic.
- **Network ACLs:** Stateless subnet-level firewalls governing traffic crossing subnet boundaries.
- **CIDR Blocks:** Define IP address ranges for networks and subnets.
- **Public and Private Subnets:** Designate subnets with or without direct internet routing capabilities.
- **VPC Flow Logs:** Capture IP traffic metadata traversing network interfaces for auditing and diagnostics.

## Virtualization Connection
Amazon VPC replaces traditional physical networking infrastructure with software-defined networking abstractions. Virtual routers, switches, and firewalls are implemented entirely through software configuration. Although these controls are virtualized, network packets still strictly rely on accurate Layer 3 addressing, valid route entries, and matching policy rules to successfully traverse from source to destination.

## Evidence Reviewed
- **VPC CIDR:** `10.42.0.0/16` associated with `vpc-0a91`.
- **Subnet CIDR & Type:** `subnet-0a91`, treated as a public subnet design layout.
- **Compute Instance:** Running EC2 instance with public IPv4 address `198.51.100.24`, listening on TCP port 80.
- **Route Table Entries:** Associated route table `rtb-0a91` contains only the local rule (`10.42.0.0/16 → local`).
- **Internet Gateway Attachment:** `igw-0a91` is successfully attached to `vpc-0a91`.
- **Security Group Rules:** `sg-0a91` permits inbound TCP port 80 traffic from any source (`0.0.0.0/0`).
- **Network ACL Rules:** Inbound TCP 80 and outbound ephemeral ports (`1024-65535`) are permitted.

## Operational Analysis
The evidence indicates a failure at Layer 3 (the Network Layer). While the instance has a public IP, an attached Internet Gateway, and fully permissive security groups and Network ACLs, the associated route table (`rtb-0a91`) lacks a default route (`0.0.0.0/0`) pointing to the Internet Gateway (`igw-0a91`). Because the route table only contains the local VPC route, incoming packets from internet clients arriving at the gateway have no defined forwarding path to reach the target instance interface, resulting in dropped traffic before instance firewalls are ever reached.

## Recommendation
The smallest supported operational correction is to modify route table `rtb-0a91` by adding an explicit default route (`0.0.0.0/0`) targeting the Internet Gateway (`igw-0a91`). Unrelated security groups, network ACLs, and instance configurations should remain untouched since they are already correctly configured.

## Escalation Notes
Standard route table updates within a development or testing VPC fall within junior operator authority. However, modifying production route tables, internet gateways, or perimeter security boundaries requires standard change management approval and peer review before execution.

## Lessons Learned
Week 9 demonstrated that attaching an Internet Gateway and assigning public IP addresses is insufficient on its own for external reachability. Validating every link in the traffic chain—especially explicit route table entries—is critical, as a missing Layer 3 route will completely neutralize otherwise correct firewall and compute configurations.

## Professional Vocabulary
- **VPC:** An isolated, user-defined virtual network privately dedicated to an AWS account.
- **Subnet:** A segmented range of IP addresses within a VPC used to isolate cloud resources.
- **Route Table:** A set of routing rules (routes) that determine where network traffic is directed.
- **Internet Gateway:** A horizontally scaled, redundant VPC component that enables communication between instances and the public internet.
- **NAT Gateway:** A managed Network Address Translation service that allows private subnet instances to initiate outbound internet traffic while blocking inbound connections.
- **Security Group:** A stateful virtual firewall that controls inbound and outbound traffic at the EC2 instance level.
- **Network ACL:** A stateless subnet-level firewall acting as an additional layer of security.
- **CIDR:** Classless Inter-Domain Routing, a methodology for allocating IP addresses and routing IP packets.
- **Public Subnet:** A subnet associated with a route table that includes a route to an Internet Gateway.
- **Private Subnet:** A subnet whose traffic is routed internally or through a NAT device, lacking direct inbound internet paths.
