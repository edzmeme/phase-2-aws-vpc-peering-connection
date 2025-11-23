# Cross-Region VPC Peering Setup

## Overview
This repository contains the configuration and documentation for a secure cross-region VPC peering connection between two AWS VPCs:
- **BaSingSe VPC** (us-east-1): 10.50.0.0/16
- **AgnaQela VPC** (us-east-2): 10.10.0.0/16

## Architecture

### Network Topology
```
┌─────────────────────────────────┐         ┌─────────────────────────────────┐
│  BaSingSe VPC (us-east-1)       │         │  AgnaQela VPC (us-east-2)       │
│  CIDR: 10.50.0.0/16             │◄───────►│  CIDR: 10.10.0.0/16             │
│  vpc-0d7a9329b45600565          │ Peering │  vpc-096cc25e1db064600          │
│                                 │         │                                 │
│  Security Group:                │         │  Security Group:                │
│  sg-0460b0fb6ae8196cd           │         │  sg-003d43d5b738433e7           │
└─────────────────────────────────┘         └─────────────────────────────────┘
         pcx-0e3da096ad63a5a1c
```

## Features

### ✅ Implemented Security Controls

1. **Cross-Region VPC Peering**
   - Active peering connection between us-east-1 and us-east-2
   - Bidirectional routing configured
   - No routing conflicts

2. **DNS Resolution Disabled**
   - Prevents cross-region DNS reconnaissance attacks
   - Both requester and accepter DNS resolution set to `false`
   - Follows principle of least privilege

3. **Restrictive Security Groups**
   - **HTTP** (port 80): Allowed from peer VPC CIDR only
   - **HTTPS** (port 443): Allowed from peer VPC CIDR only
   - **ICMP** (ping): Allowed for connectivity testing
   - All other traffic: Denied by default

4. **Validated Routing**
   - Each VPC only routes to the peer VPC, not to itself
   - No self-referencing routes (routing conflicts eliminated)
   - Routes automatically distributed to all route tables

## Resource Details

### VPC Information

| Resource | Region | VPC ID | CIDR Block | Purpose |
|----------|--------|--------|------------|---------|
| BaSingSe | us-east-1 | vpc-0d7a9329b45600565 | 10.50.0.0/16 | Primary application VPC |
| AgnaQela | us-east-2 | vpc-096cc25e1db064600 | 10.10.0.0/16 | Secondary/DR VPC |

### Peering Connection

| Attribute | Value |
|-----------|-------|
| Connection ID | pcx-0e3da096ad63a5a1c |
| Status | Active |
| Requester Region | us-east-1 |
| Accepter Region | us-east-2 |
| DNS Resolution | Disabled (both sides) |

### Security Groups

| VPC | Security Group ID | Allowed Traffic |
|-----|-------------------|-----------------|
| BaSingSe | sg-0460b0fb6ae8196cd | HTTP/HTTPS/ICMP from 10.10.0.0/16 |
| AgnaQela | sg-003d43d5b738433e7 | HTTP/HTTPS/ICMP from 10.50.0.0/16 |

## Setup Instructions

### Prerequisites
- AWS CLI configured with appropriate credentials
- Access to both us-east-1 and us-east-2 regions
- IAM permissions for EC2, VPC operations

### Manual Setup Steps

#### 1. Create VPC Peering Connection
```bash
# Create peering connection
PEER_ID=$(aws ec2 create-vpc-peering-connection \
    --vpc-id vpc-0d7a9329b45600565 \
    --peer-vpc-id vpc-096cc25e1db064600 \
    --peer-region us-east-2 \
    --region us-east-1 \
    --query 'VpcPeeringConnection.VpcPeeringConnectionId' \
    --output text)

# Accept peering connection from us-east-2
aws ec2 accept-vpc-peering-connection \
    --vpc-peering-connection-id $PEER_ID \
    --region us-east-2
```

#### 2. Configure Routes
```bash
# Add routes in us-east-1 to us-east-2 VPC
for rt in $(aws ec2 describe-route-tables --filters "Name=vpc-id,Values=vpc-0d7a9329b45600565" --query 'RouteTables[].RouteTableId' --output text --region us-east-1); do
    aws ec2 create-route --route-table-id $rt --destination-cidr-block 10.10.0.0/16 --vpc-peering-connection-id $PEER_ID --region us-east-1
done

# Add routes in us-east-2 to us-east-1 VPC
for rt in $(aws ec2 describe-route-tables --filters "Name=vpc-id,Values=vpc-096cc25e1db064600" --query 'RouteTables[].RouteTableId' --output text --region us-east-2); do
    aws ec2 create-route --route-table-id $rt --destination-cidr-block 10.50.0.0/16 --vpc-peering-connection-id $PEER_ID --region us-east-2
done
```

#### 3. Disable DNS Resolution (Security Hardening)
```bash
# Disable DNS resolution for requester (us-east-1)
aws ec2 modify-vpc-peering-connection-options \
    --vpc-peering-connection-id $PEER_ID \
    --requester-peering-connection-options AllowDnsResolutionFromRemoteVpc=false \
    --region us-east-1

# Disable DNS resolution for accepter (us-east-2)
aws ec2 modify-vpc-peering-connection-options \
    --vpc-peering-connection-id $PEER_ID \
    --accepter-peering-connection-options AllowDnsResolutionFromRemoteVpc=false \
    --region us-east-2
```

#### 4. Create Security Groups
```bash
# BaSingSe security group (us-east-1)
SG_BASSINGSE=$(aws ec2 create-security-group \
    --group-name "cross-region-peering-bassingse" \
    --description "Cross-region peering security group for BaSingSe" \
    --vpc-id vpc-0d7a9329b45600565 \
    --region us-east-1 \
    --query 'GroupId' \
    --output text)

aws ec2 authorize-security-group-ingress --group-id $SG_BASSINGSE --protocol tcp --port 443 --cidr 10.10.0.0/16 --region us-east-1
aws ec2 authorize-security-group-ingress --group-id $SG_BASSINGSE --protocol tcp --port 80 --cidr 10.10.0.0/16 --region us-east-1
aws ec2 authorize-security-group-ingress --group-id $SG_BASSINGSE --protocol icmp --port -1 --cidr 10.10.0.0/16 --region us-east-1

# AgnaQela security group (us-east-2)
SG_AGNAQELA=$(aws ec2 create-security-group \
    --group-name "cross-region-peering-agnaqela" \
    --description "Cross-region peering security group for AgnaQela" \
    --vpc-id vpc-096cc25e1db064600 \
    --region us-east-2 \
    --query 'GroupId' \
    --output text)

aws ec2 authorize-security-group-ingress --group-id $SG_AGNAQELA --protocol tcp --port 443 --cidr 10.50.0.0/16 --region us-east-2
aws ec2 authorize-security-group-ingress --group-id $SG_AGNAQELA --protocol tcp --port 80 --cidr 10.50.0.0/16 --region us-east-2
aws ec2 authorize-security-group-ingress --group-id $SG_AGNAQELA --protocol icmp --port -1 --cidr 10.50.0.0/16 --region us-east-2
```

## Verification

### Verify Peering Connection Status
```bash
aws ec2 describe-vpc-peering-connections \
    --vpc-peering-connection-ids pcx-0e3da096ad63a5a1c \
    --query 'VpcPeeringConnections[0].{Status:Status.Code,RequesterDNS:RequesterVpcInfo.PeeringOptions.AllowDnsResolutionFromRemoteVpc,AccepterDNS:AccepterVpcInfo.PeeringOptions.AllowDnsResolutionFromRemoteVpc}' \
    --output table
```

### Verify Routes
```bash
# Check us-east-1 routes
aws ec2 describe-route-tables \
    --filters "Name=route.vpc-peering-connection-id,Values=pcx-0e3da096ad63a5a1c" \
    --region us-east-1 \
    --output table

# Check us-east-2 routes
aws ec2 describe-route-tables \
    --filters "Name=route.vpc-peering-connection-id,Values=pcx-0e3da096ad63a5a1c" \
    --region us-east-2 \
    --output table
```

### Test Connectivity
```bash
# From an instance in BaSingSe VPC, test ping to AgnaQela instance
ping <private-ip-in-agnaqela-vpc>

# Test HTTP connectivity
curl http://<private-ip-in-agnaqela-vpc>
```

## Security Best Practices

### Implemented
- ✅ DNS resolution disabled to prevent reconnaissance
- ✅ Security groups with least privilege access
- ✅ No routing conflicts (validated)
- ✅ Cross-region traffic restricted to specific ports

### Production Recommendations
1. **Remove ICMP after testing**
   ```bash
   aws ec2 revoke-security-group-ingress --group-id sg-0460b0fb6ae8196cd --protocol icmp --port -1 --cidr 10.10.0.0/16 --region us-east-1
   aws ec2 revoke-security-group-ingress --group-id sg-003d43d5b738433e7 --protocol icmp --port -1 --cidr 10.50.0.0/16 --region us-east-2
   ```

2. **Enable VPC Flow Logs for monitoring**
   ```bash
   aws ec2 create-flow-logs \
       --resource-type VPC \
       --resource-ids vpc-0d7a9329b45600565 \
       --traffic-type ALL \
       --log-destination-type cloud-watch-logs \
       --log-group-name VPC-Peering-FlowLogs
   ```

3. **Restrict security group rules to specific ports needed**
   - Only allow the exact ports your applications require
   - Avoid broad CIDR ranges where possible

4. **Implement Network ACLs for additional layer of security**
   - Add subnet-level controls beyond security groups

5. **Regular security audits**
   - Review security group rules quarterly
   - Monitor VPC Flow Logs for anomalies
   - Validate routing tables haven't changed

## Troubleshooting

### Common Issues

#### Connectivity Failures
**Problem:** Cannot ping between VPCs  
**Solution:**
1. Verify peering connection status is "Active"
2. Check routes exist in both VPCs pointing to peer CIDR
3. Confirm security groups allow ICMP from peer VPC CIDR
4. Verify instances are in subnets with proper route table associations

#### Routing Conflicts
**Problem:** Routes pointing to local VPC CIDR via peering connection  
**Solution:**
```bash
# Check for self-referencing routes
aws ec2 describe-route-tables --filters "Name=vpc-id,Values=vpc-0d7a9329b45600565" --query 'RouteTables[].Routes[?DestinationCidrBlock==`10.50.0.0/16` && VpcPeeringConnectionId!=null]' --region us-east-1

# Delete if found
aws ec2 delete-route --route-table-id <rtb-id> --destination-cidr-block 10.50.0.0/16
```

#### DNS Resolution Issues
**Problem:** Cannot resolve private DNS names across regions  
**Expected Behavior:** DNS resolution is intentionally disabled for security. Use IP addresses for cross-region communication.

## Cost Considerations

### Data Transfer Costs
- **Within same region:** Free
- **Cross-region:** $0.01 - $0.02 per GB (varies by region pair)
- us-east-1 to us-east-2: ~$0.01/GB

### Estimate Monthly Costs
- 1 TB cross-region transfer: ~$10/month
- VPC Peering connection: No additional charge
- VPC Flow Logs (if enabled): ~$0.50/GB ingested

## Maintenance

### Regular Tasks
- [ ] Monthly security group audit
- [ ] Quarterly routing table review
- [ ] Monitor VPC Flow Logs for anomalies
- [ ] Review and update documentation

### Updating Routes
If you add new subnets to either VPC, ensure routes are added to their associated route tables:
```bash
aws ec2 create-route \
    --route-table-id <new-route-table-id> \
    --destination-cidr-block <peer-vpc-cidr> \
    --vpc-peering-connection-id pcx-0e3da096ad63a5a1c
```

## Known Limitations

1. **DNS Resolution:** Disabled for security - use IP addresses for cross-region communication
2. **Cross-Region Bandwidth:** Subject to inter-region transfer costs
3. **Transitive Peering:** Not supported - each VPC pair requires separate peering connection
4. **Maximum Peering Connections:** 125 active peering connections per VPC

## Rollback Plan

To remove the peering connection:
```bash
# 1. Delete routes
aws ec2 delete-route --route-table-id <rtb-id> --destination-cidr-block <peer-cidr>

# 2. Delete security groups
aws ec2 delete-security-group --group-id sg-0460b0fb6ae8196cd --region us-east-1
aws ec2 delete-security-group --group-id sg-003d43d5b738433e7 --region us-east-2

# 3. Delete peering connection
aws ec2 delete-vpc-peering-connection --vpc-peering-connection-id pcx-0e3da096ad63a5a1c
```

## References

- [AWS VPC Peering Documentation](https://docs.aws.amazon.com/vpc/latest/peering/)
- [Cross-Region VPC Peering](https://docs.aws.amazon.com/vpc/latest/peering/working-with-vpc-peering.html)
- [Security Groups Best Practices](https://docs.aws.amazon.com/vpc/latest/userguide/VPC_SecurityGroups.html)

## Contributing

This infrastructure was configured on 2025-06-25. For updates or modifications, please:
1. Test changes in a non-production environment
2. Update this documentation
3. Validate routing and security group configurations
4. Submit changes for review

## Support

For issues or questions:
- Review troubleshooting section above
- Check AWS VPC service health dashboard
- Verify IAM permissions for VPC operations

---

**Last Updated:** 2025-06-25  
**Configuration Version:** 1.0  
**Status:** Production Ready
