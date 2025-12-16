# DRS Infrastructure CloudFormation Template Cost Analysis Estimate Report

## Service Overview

AWS Elastic Disaster Recovery (DRS) provides continuous replication of your on-premises or cloud-based servers into AWS. This project uses multiple AWS services to support DRS operations including storage replication, networking, and compute resources for disaster recovery drills. The service follows a pay-as-you-go pricing model, making it cost-effective for various workloads.

## Pricing Model

This cost analysis estimate is based on the following pricing model:
- **ON DEMAND** pricing (pay-as-you-go) unless otherwise specified
- Standard service configurations without reserved capacity or savings plans
- No caching or optimization techniques applied

## Key Assumptions Summary

**Pricing & Region:**
- US-West-2 region, ON DEMAND pricing
- 730 hours/month (24/7 operation)

**Compute & Licensing:**
- Windows EC2 t3.small with full licensing ($0.0392/hour)
- Alternative: BYOL licensing available ($0.0208/hour)

**Networking:**
- Route 53 Resolver: 2 network interfaces required (multi-AZ)
- VPN Connection: $36/month industry standard
- VPC Endpoints: $7.20/month per interface endpoint
- 8 VPC endpoints total (S3, SSM, EC2Messages, etc.)

**Usage:**
- Minimal DNS queries, API calls, and data transfer
- Single secret in Secrets Manager
- One private hosted zone
- 50GB Ubuntu server replication (continuous)
- DRS replication server (only during drills/tests)

## Limitations and Exclusions

- Data transfer costs between regions
- VPN data processing charges
- Route 53 hosted zone query charges beyond minimal usage
- CloudWatch logs storage and ingestion costs
- Secrets Manager API request charges beyond minimal usage
- DRS staging area compute costs (only incurred during drills/tests)
- Data transfer costs for initial replication and ongoing changes
- Additional EBS snapshots created by DRS

## Cost Breakdown

### Unit Pricing Details

| Service | Resource Type | Unit | Price | Free Tier |
|---------|--------------|------|-------|------------|
| EC2 Windows Instance (t3.small) | Windows Byol | hour | $0.0208 | No free tier for Windows instances |
| EC2 Windows Instance (t3.small) | Windows Licensed | hour | $0.0392 | No free tier for Windows instances |
| VPN Connection | Connection | month per connection | $36.00 | No free tier for VPN connections |
| VPC Endpoints (8 Interface Endpoints) | Interface Endpoint | month per endpoint | $7.20 | No free tier for VPC endpoints |
| Route 53 Resolver Inbound Endpoint | Network Interface | hour per interface | $0.125 | No free tier for Route 53 Resolver |
| Route 53 Private Hosted Zone | Hosted Zone | month per zone | $0.50 | No free tier for private hosted zones |
| AWS Secrets Manager | Secret | month per secret | $0.40 | No free tier for Secrets Manager |
| AWS DRS Replication | Replication Hour | hour per server | $0.0280 | No free tier for DRS replication |
| EBS GP3 Storage (Replicated) | Storage | GB per month | $0.08 | No free tier for EBS storage |
| DRS Recovery Instance (m5.xlarge Linux) | Instance Hour | hour (test only) | $0.192 | No free tier for EC2 instances |
| VPC and Networking (Free Resources) | Vpc Resources | 1 unit | $0.00 | VPC, subnets, route tables, and security groups are free |

### Cost Calculation

| Service | Usage | Calculation | Monthly Cost |
|---------|-------|-------------|-------------|
| EC2 Windows Instance (t3.small) | 1 instance running 730 hours/month (Hours: 730 hours/month) | $0.0392/hour � 730 hours = $28.62 (assuming Windows with license) | $28.62 |
| VPN Connection | 1 Site-to-Site VPN connection (Connections: 1 VPN connection) | $36.00 flat monthly fee per VPN connection | $36.00 |
| VPC Endpoints (8 Interface Endpoints) | 8 interface endpoints (S3, SSM, EC2Messages, etc.) (Endpoints: 8 interface endpoints) | $7.20/month � 8 endpoints = $57.60 | $57.60 |
| Route 53 Resolver Inbound Endpoint | 1 inbound endpoint with 2 network interfaces (multi-AZ) (Interfaces: 2 network interfaces � 730 hours) | $0.125/hour � 2 interfaces � 730 hours = $182.50 | $182.50 |
| Route 53 Private Hosted Zone | 1 private hosted zone for DRS endpoint (Zones: 1 private hosted zone) | $0.50/month per private hosted zone | $0.50 |
| AWS Secrets Manager | 1 secret storing RDP credentials (Secrets: 1 secret) | $0.40/month per secret | $0.40 |
| AWS DRS Replication | 1 Ubuntu server replicated 24/7 (Hours: 730 hours/month × 1 server) | $0.0280/hour × 730 hours = $20.44 | $20.44 |
| EBS GP3 Storage (Replicated) | 50GB replicated storage (Storage: 50 GB replicated continuously) | $0.08/GB-month × 50 GB = $4.00 | $4.00 |
| DRS Recovery Instance (m5.xlarge Linux) | Used only during DR tests - estimated 4 hours/month (Hours: 4 hours/month for testing) | $0.192/hour × 4 hours = $0.77 | $0.77 |
| VPC and Networking (Free Resources) | 1 VPC, 6 subnets, 2 route tables, 3 security groups (Vpc: 1 VPC, 6 subnets, 2 route tables, 3 security groups) | No charges for basic VPC networking components | $0.00 |
| **Total** | **All services** | **Sum of all calculations** | **$330.91/month** |

### Free Tier

Free tier information by service:
- **EC2 Windows Instance (t3.small)**: No free tier for Windows instances
- **VPN Connection**: No free tier for VPN connections
- **VPC Endpoints (8 Interface Endpoints)**: No free tier for VPC endpoints
- **Route 53 Resolver Inbound Endpoint**: No free tier for Route 53 Resolver
- **Route 53 Private Hosted Zone**: No free tier for private hosted zones
- **AWS Secrets Manager**: No free tier for Secrets Manager
- **AWS DRS Replication**: No free tier for DRS replication
- **EBS GP3 Storage (Replicated)**: No free tier for EBS storage
- **DRS Recovery Instance (m5.xlarge Linux)**: No free tier for EC2 instances
- **VPC and Networking (Free Resources)**: VPC, subnets, route tables, and security groups are free

### Key Cost Factors

- **EC2 Windows Instance (t3.small)**: 1 instance running 730 hours/month
- **VPN Connection**: 1 Site-to-Site VPN connection
- **VPC Endpoints (8 Interface Endpoints)**: 8 interface endpoints (S3, SSM, EC2Messages, etc.)
- **Route 53 Resolver Inbound Endpoint**: 1 inbound endpoint with 2 network interfaces (multi-AZ)
- **Route 53 Private Hosted Zone**: 1 private hosted zone for DRS endpoint
- **AWS Secrets Manager**: 1 secret storing RDP credentials
- **AWS DRS Replication**: 1 Ubuntu server replicated 24/7
- **EBS GP3 Storage (Replicated)**: 50GB replicated storage
- **DRS Recovery Instance (m5.xlarge Linux)**: Used only during DR tests - estimated 4 hours/month
- **VPC and Networking (Free Resources)**: 1 VPC, 6 subnets, 2 route tables, 3 security groups

## AWS DRS Specific Cost Analysis

### DRS Replication Costs

**Primary Ongoing Costs:**
- **DRS Replication Service**: $0.0280 per replication-hour per server
- **Replicated EBS Storage**: $0.08 per GB-month for GP3 storage
- **Replication Server**: $0.0208 per hour (t3.small Linux) - runs continuously
- **Recovery Instance**: $0.192 per hour (m5.xlarge Linux) - only during DR tests

**Key Assumptions for DRS Costs:**
- 1 Ubuntu server (50GB disk) being replicated continuously
- Recovery instance (m5.xlarge) runs only during monthly DR tests (estimated 4 hours/month)
- No additional data transfer costs for initial sync (included in replication cost)
- Standard GP3 storage performance (3,000 IOPS, 125 MB/s throughput)
- No additional snapshots beyond DRS-managed replication

**Monthly DRS Cost Breakdown:**
- Replication service: 730 hours × $0.0280 = $20.44
- Replicated storage: 50 GB × $0.08 = $4.00
- Recovery instance testing: 4 hours × $0.192 = $0.77
- **Total DRS costs: $25.21/month**

### DRS Cost Scaling Scenarios

| Servers | Storage per Server | Monthly DRS Cost | Annual DRS Cost |
|---------|-------------------|------------------|------------------|
| 1 | 50GB | $25.21 | $302.52 |
| 3 | 50GB | $75.63 | $907.56 |
| 5 | 100GB | $166.05 | $1,992.60 |
| 10 | 100GB | $332.10 | $3,985.20 |

*Note: Costs exclude data transfer and additional testing beyond 4 hours/month*

## Detailed Cost Analysis

### Pricing Model

ON DEMAND


### Exclusions

- Data transfer costs between regions
- VPN data processing charges
- Route 53 hosted zone query charges beyond minimal usage
- CloudWatch logs storage and ingestion costs
- Secrets Manager API request charges beyond minimal usage

### Recommendations

#### Immediate Actions

- Consider using Windows BYOL licensing to reduce EC2 costs from $28.62 to $15.18/month
- Evaluate if Route 53 Resolver is necessary - could save $182.50/month if DNS forwarding is used instead
- VPC endpoints required for complete private connectivity without internet access
#### Capacity Assurance for Critical Recovery Instances

**On-Demand Capacity Reservations (ODCR)** guarantee EC2 capacity availability in specific Availability Zones for disaster recovery scenarios.

**How ODCR Works:**
- Reserve capacity for specific instance types (e.g., m5.xlarge for recovery instances)
- Pay On-Demand rates whether capacity is used or unused ($0.192/hour continuously)
- No double billing - when instances run in reserved capacity, you pay the same single On-Demand rate
- Can cancel anytime (no long-term commitment like Reserved Instances)

**Cost Impact for DRS Recovery Instances:**
- **Without ODCR**: $0.77/month (4 hours testing) + capacity availability risk
- **With ODCR**: $140.16/month (730 hours reserved) + guaranteed capacity

**Business Consideration:**
ODCR provides insurance against capacity constraints during actual disasters when regional EC2 capacity may be limited. The $139/month premium ensures recovery instances can launch when needed most.

**Recommendation:**
Evaluate ODCR for business-critical workloads where guaranteed capacity availability outweighs the cost of continuous reservation. Consider for regulatory compliance requirements or when disaster recovery SLAs mandate capacity assurance.

#### Best Practices

- Implement comprehensive tagging strategy with Project: DRS-VMware for cost tracking
- Set up AWS Budgets with alerts for monthly cost thresholds
- Monitor VPN data transfer charges which are not included in base connection fee
- Exclude non-essential disks or files from DRS replication to reduce storage costsith alerts for monthly cost thresholds
- Monitor VPN data transfer charges which are not included in base connection fee
- Consider On-Demand Capacity Reservations for critical DR instances
- Use CloudWatch to monitor resource utilization and right-size instances
- Exclude non-essential disks or files from DRS replication to reduce storage costs



## Conclusion

By following the recommendations in this report, you can optimize your DRS Infrastructure CloudFormation Template costs while maintaining performance and reliability. Regular monitoring and adjustment of your usage patterns will help ensure cost efficiency as your workload evolves.
