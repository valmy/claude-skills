# Pulumi Alibaba Cloud Best Practices (Python)

## Provider Configuration

```python
import pulumi
import pulumi_alicloud as alicloud

# Use ESC for credentials via OIDC

# Multi-region deployments with explicit providers
singapore_provider = alicloud.Provider("singapore", region="ap-southeast-1")
us_provider = alicloud.Provider("us", region="us-west-1")

# Use provider with resources
vpc = alicloud.vpc.Network(
    "vpc",
    vpc_name="my-vpc",
    cidr_block="172.16.0.0/16",
    opts=pulumi.ResourceOptions(provider=singapore_provider),
)
```

## Essential Resources

### VPC - Network Pattern

```python
import pulumi
import pulumi_alicloud as alicloud

vpc = alicloud.vpc.Network(
    "vpc",
    vpc_name=f"vpc-{pulumi.get_project()}-{pulumi.get_stack()}",
    cidr_block="172.16.0.0/16",
    region_id="ap-southeast-1",  # Using Singapore region
    description="VPC for application infrastructure",

    # Tags for resource organization
    tags={
        "Environment": pulumi.get_stack(),
        "Project": pulumi.get_project(),
        "ManagedBy": "Pulumi",
    },
)

# Get available zones for the region
zones = alicloud.get_zones(
    available_resource_creation="VSwitch",
    region_id="ap-southeast-1",  # Using Singapore region
)

# Create subnets in multiple zones for high availability
vswitch = alicloud.vpc.Switch(
    "vswitch",
    vswitch_name=f"subnet-{pulumi.get_stack()}",
    region_id="ap-southeast-1",  # Using Singapore region
    cidr_block="172.16.0.0/24",
    zone_id=zones.zones[0].id,
    vpc_id=vpc.id,

    tags={
        "Environment": pulumi.get_stack(),
        "Purpose": "public",
    },
)

# Security group for network access control
security_group = alicloud.ecs.SecurityGroup(
    "sg",
    security_group_name=f"sg-{pulumi.get_stack()}",
    region_id="ap-southeast-1",  # Using Singapore region
    vpc_id=vpc.id,
    description="Security group for application servers",

    tags={
        "Environment": pulumi.get_stack(),
    },
)

# Add security group rules
ssh_rule = alicloud.ecs.SecurityGroupRule(
    "ssh-ingress",
    type="ingress",
    region_id="ap-southeast-1",  # Using Singapore region
    ip_protocol="tcp",
    nic_type="intranet",
    policy="accept",
    port_range="22/22",
    priority=1,
    security_group_id=security_group.id,
    cidr_ip="0.0.0.0/0",  # Restrict this in production
)

http_rule = alicloud.ecs.SecurityGroupRule(
    "http-ingress",
    type="ingress",
    region_id="ap-southeast-1",  # Using Singapore region
    ip_protocol="tcp",
    nic_type="intranet",
    policy="accept",
    port_range="80/80",
    priority=100,
    security_group_id=security_group.id,
    cidr_ip="0.0.0.0/0",
)

pulumi.export("vpc_id", vpc.id)
pulumi.export("vswitch_id", vswitch.id)
pulumi.export("security_group_id", security_group.id)
```

### ECS - Elastic Compute Service Pattern

```python
import pulumi
import pulumi_alicloud as alicloud

config = pulumi.Config()
is_prod = pulumi.get_stack() == "prod"

# Create KMS key for disk encryption
kms_key = alicloud.kms.Key(
    "encryption-key",
    description="Key for encrypting ECS data disks",
    pending_window_in_days=7,
    status="Enabled",
    
    tags={
        "Environment": pulumi.get_stack(),
    },
)

# ECS instance with security and encryption
instance = alicloud.ecs.Instance(
    "web-server",
    instance_name=f"web-{pulumi.get_project()}-{pulumi.get_stack()}",
    description="Web server instance",

    # Instance configuration
    instance_type="ecs.c6.large" if is_prod else "ecs.t6.small",
    image_id="ubuntu_20_04_x64_20G_alibase_20230713.vhd",  # Use latest image
    region_id="ap-southeast-1",  # Using Singapore region

    # Network configuration
    vswitch_id=vswitch.id,
    security_groups=[security_group.id],

    # Storage configuration
    system_disk_category="cloud_essd",
    system_disk_size=40,

    # Security and encryption
    key_pair_name="my-keypair",  # SSH key pair name
    deletion_protection=is_prod,

    # Billing
    instance_charge_type="PostPaid",  # More flexible for dev environments
    internet_max_bandwidth_out=10 if is_prod else 5,

    # Tags
    tags={
        "Environment": pulumi.get_stack(),
        "Role": "web-server",
        "ManagedBy": "Pulumi",
    },

    # Data disks with encryption
    data_disks=[{
        "name": "data-disk",
        "size": 100,
        "category": "cloud_essd",
        "encrypted": True,
        "kms_key_id": kms_key.id,
        "delete_with_instance": True,
    }],
)

pulumi.export("instance_id", instance.id)
pulumi.export("instance_private_ip", instance.private_ip)
pulumi.export("instance_public_ip", instance.public_ip)
```

### RDS - Database Pattern

```python
import pulumi
import pulumi_alicloud as alicloud

config = pulumi.Config()
is_prod = pulumi.get_stack() == "prod"

# RDS subnet group
rds_subnet_group = alicloud.rds.Account(
    "rds-account",  # Note: This is a placeholder - actual RDS setup requires more configuration
    region_id="ap-southeast-1",  # Using Singapore region
)

# RDS instance with security and backup
rds_instance = alicloud.rds.Instance(
    "postgres-db",
    instance_name=f"db-{pulumi.get_project()}-{pulumi.get_stack()}",

    # Engine and version
    engine="PostgreSQL",
    engine_version="12.0",  # Use latest stable version

    # Instance configuration
    instance_type="rds.pg.x4.large" if is_prod else "rds.pg.x2.small",
    instance_storage=100 if is_prod else 20,  # GB

    # Network configuration
    vpc_id=vpc.id,
    vswitch_id=vswitch.id,
    zone_id=zones.zones[0].id,
    # Specify region for RDS instance
    region_id="ap-southeast-1",  # Using Singapore region

    # Security
    security_ips=["127.0.0.1"],  # Restrict access in production
    db_instance_storage_type="cloud_essd",  # High performance storage
    encryption_key=kms_key.id,  # Use KMS encryption

    # Backup and maintenance
    backup_retention_period=7,
    maintain_time="03:00Z-04:00Z",  # Maintenance window

    # Billing
    instance_charge_type="PostPaid",

    # High availability (production only)
    category="HighAvailability" if is_prod else "Basic",

    # Security
    deletion_protection=is_prod,

    # Tags
    tags={
        "Environment": pulumi.get_stack(),
        "Tier": "database",
        "ManagedBy": "Pulumi",
    },
)

# Create database within the instance
database = alicloud.rds.Database(
    "app-database",
    instance_id=rds_instance.id,
    region_id="ap-southeast-1",  # Using Singapore region
    name="app_db",
    character_set="utf8mb4",
)

# Create database account
account = alicloud.rds.Account(
    "app-account",
    instance_id=rds_instance.id,
    region_id="ap-southeast-1",  # Using Singapore region
    account_name="app_user",
    account_password=config.require_secret("db_password"),
    account_type="Normal",
)

pulumi.export("rds_connection_string", rds_instance.connection_string)
pulumi.export("rds_port", rds_instance.port)
```

### Object Storage Service (OSS) - Secure Bucket Pattern

```python
import pulumi
import pulumi_alicloud as alicloud

oss_bucket = alicloud.oss.Bucket(
    "data-bucket",
    bucket=f"data-{pulumi.get_project()}-{pulumi.get_stack()}",
    region_id="ap-southeast-1",  # Using Singapore region

    # Storage class for cost optimization
    storage_class="Standard",

    # Server-side encryption
    server_side_encryption_rule={
        "sse_algorithm": "KMS",
        "kms_master_key_id": kms_key.id,
    },

    # Versioning for data protection
    versioning="Enabled",

    # Access control
    acl="private",

    # Lifecycle rules for cost optimization
    lifecycle_rule=[
        {
            "id": "transition-to-ia",
            "status": "Enabled",
            "prefix": "logs/",
            "actions": [
                {
                    "type": "Transition",
                    "storage_class": "IA",
                    "transition_delayed_days": 30,
                },
                {
                    "type": "Transition",
                    "storage_class": "Archive",
                    "transition_delayed_days": 90,
                },
                {
                    "type": "Expiration",
                    "expiration_delayed_days": 365,
                },
            ],
        },
    ],

    # Tags
    tags={
        "Environment": pulumi.get_stack(),
        "Purpose": "data-storage",
        "ManagedBy": "Pulumi",
    },
)

# Bucket policy to deny insecure transport
bucket_policy = alicloud.oss.BucketPolicy(
    "secure-policy",
    bucket=oss_bucket.id,
    region_id="ap-southeast-1",  # Using Singapore region
    policy=oss_bucket.id.apply(lambda id: f"""{{
        "Version": "1",
        "Statement": [
            {{
                "Effect": "Deny",
                "Principal": "*",
                "Action": "oss:GetObject",
                "Resource": "acs:oss:*:*:{id}/*",
                "Condition": {{
                    "Bool": {{
                        "acs:SecureTransport": "false"
                    }}
                }}
            }}
        ]
    }}"""),
)

pulumi.export("oss_bucket_name", oss_bucket.bucket)
pulumi.export("oss_bucket_domain", oss_bucket.extranet_endpoint)
```

### Server Load Balancer (SLB) - Application Load Balancer Pattern

```python
import pulumi
import pulumi_alicloud as alicloud

# Note: SLB is deprecated in favor of Application Load Balancer
alb = alicloud.alb.LoadBalancer(
    "application-lb",
    load_balancer_name=f"alb-{pulumi.get_project()}-{pulumi.get_stack()}",
    address_type="Internet",
    region_id="ap-southeast-1",  # Using Singapore region
    vpc_id=vpc.id,
    vswitch_ids=[vswitch.id],

    # Load balancer specification
    load_balancer_edition="Standard",
    load_balancer_billing_config={
        "pay_type": "PayAsYouYou",
    },

    # Security
    deletion_protection_enabled=True,

    # Tags
    tags={
        "Environment": pulumi.get_stack(),
        "Purpose": "load-balancer",
        "ManagedBy": "Pulumi",
    },
)

# Listener for HTTP traffic
listener = alicloud.alb.Listener(
    "http-listener",
    listener_name=f"http-{pulumi.get_stack()}",
    region_id="ap-southeast-1",  # Using Singapore region
    load_balancer_id=alb.id,
    protocol="HTTP",
    port=80,

    # Default actions
    default_actions=[{
        "type": "ForwardGroup",
        "forward_group_id": server_group.id,
    }],
)

# Server group for backend instances
server_group = alicloud.alb.ServerGroup(
    "backend-servers",
    server_group_name=f"backend-{pulumi.get_stack()}",
    region_id="ap-southeast-1",  # Using Singapore region
    load_balancer_id=alb.id,

    # Backend servers
    servers=[{
        "server_id": instance.id,
        "server_type": "Ecs",
        "weight": 100,
        "port": 80,
    }],
)

pulumi.export("alb_dns_name", alb.dns_name)
pulumi.export("alb_id", alb.id)
```

## Security Best Practices

### RAM - Resource Access Management

```python
import pulumi
import pulumi_alicloud as alicloud

# Create a custom policy for minimal permissions
custom_policy = alicloud.ram.Policy(
    "app-policy",
    policy_name=f"policy-{pulumi.get_project()}-{pulumi.get_stack()}",
    region_id="ap-southeast-1",  # Using Singapore region
    description="Custom policy for application access",
    policy_document=oss_bucket.id.apply(lambda bucket_id: f"""{{
        "Version": "1",
        "Statement": [
            {{
                "Effect": "Allow",
                "Action": [
                    "oss:GetObject",
                    "oss:PutObject"
                ],
                "Resource": [
                    "acs:oss:*:*:{bucket_id}/*"
                ]
            }}
        ]
    }}"""),
    type="Custom",
)

# Create a RAM role for ECS instances
role = alicloud.ram.Role(
    "ecs-role",
    role_name=f"role-{pulumi.get_project()}-{pulumi.get_stack()}",
    region_id="ap-southeast-1",  # Using Singapore region
    document="""{
        "Statement": [
            {
                "Action": "sts:AssumeRole",
                "Effect": "Allow",
                "Principal": {
                    "Service": [
                        "ecs.aliyuncs.com"
                    ]
                }
            }
        ],
        "Version": "1"
    }""",
    description="Role for ECS instances",
)

# Attach policy to role
policy_attachment = alicloud.ram.Attachment(
    "attach-policy",
    role_name=role.name,
    region_id="ap-southeast-1",  # Using Singapore region
    policy_name=custom_policy.name,
    policy_type=custom_policy.type,
)
```

### KMS - Key Management Service

```python
import pulumi_alicloud as alicloud

# Already shown in previous examples, but important for security
kms_key = alicloud.kms.Key(
    "encryption-key",
    description="Key for encrypting sensitive data",
    region_id="ap-southeast-1",  # Using Singapore region
    pending_window_in_days=7,
    status="Enabled",

    tags={
        "Environment": pulumi.get_stack(),
        "Purpose": "encryption",
    },
)
```

## Auto-Scaling

```python
import pulumi_alicloud as alicloud

# Auto Scaling Group
scaling_group = alicloud.ess.ScalingGroup(
    "web-scaling-group",
    scaling_group_name=f"scaling-{pulumi.get_project()}-{pulumi.get_stack()}",
    max_size=10,
    min_size=2,
    region_id="ap-southeast-1",  # Using Singapore region
    vswitch_ids=[vswitch.id],
    removal_policies=["OldestInstance", "NewestInstance"],

    tags={
        "Environment": pulumi.get_stack(),
        "Purpose": "auto-scaling",
    },
)

# Scaling configuration
scaling_config = alicloud.ess.ScalingConfiguration(
    "web-config",
    scaling_group_id=scaling_group.id,
    region_id="ap-southeast-1",  # Using Singapore region
    image_id="ubuntu_20_04_x64_20G_alibase_20230713.vhd",
    instance_type="ecs.c6.large",
    security_group_id=security_group.id,
    key_pair_name="my-keypair",

    tags={
        "Environment": pulumi.get_stack(),
    },
)

# Scaling rule
scaling_rule = alicloud.ess.ScalingRule(
    "cpu-scale-out",
    scaling_group_id=scaling_group.id,
    region_id="ap-southeast-1",  # Using Singapore region
    adjustment_type="QuantityChangeInCapacity",
    adjustment_value=2,
    scaling_rule_name="scale-out-rule",
    cooldown=300,
)
```

## Tagging Strategy

```python
import pulumi

# Standard tags for all Alibaba Cloud resources
default_tags = {
    "Environment": pulumi.get_stack(),
    "Project": pulumi.get_project(),
    "ManagedBy": "Pulumi",
    "Team": "engineering",  # Customize based on your organization
    "BusinessUnit": "web-services",  # Customize based on your organization
}

# Register transformation to add tags to all resources
def add_tags(args: pulumi.ResourceTransformationArgs):
    if hasattr(args.props, 'tags') and args.props.tags is not None:
        # Merge default tags with existing tags, preferring existing tags
        merged_tags = {**default_tags, **args.props.tags}
        args.props.tags = merged_tags
    elif hasattr(args.props, 'tags'):
        # If tags property exists but is None, set it to default tags
        args.props.tags = default_tags
    return pulumi.ResourceTransformationResult(args.props, args.opts)

pulumi.runtime.register_stack_transformation(add_tags)
```

## Cost Optimization Tips

1. **Use Reserved Instances**: For predictable workloads, consider reserved instances for significant cost savings
2. **Right-size Resources**: Regularly review and adjust instance sizes based on actual usage
3. **Use Spot Instances**: For fault-tolerant workloads, spot instances can provide up to 90% discount
4. **Implement Lifecycle Policies**: Use OSS lifecycle rules to transition data to lower-cost storage classes
5. **Monitor Resource Utilization**: Use CloudMonitor to identify underutilized resources
6. **Clean Up Unused Resources**: Regularly remove unused snapshots, images, and other resources
7. **Use Auto-Scaling**: Implement auto-scaling to match capacity with demand

## Multi-Region Deployment

For global applications, consider deploying resources across multiple regions:

```python
import pulumi
import pulumi_alicloud as alicloud

# Create resources in multiple regions
singapore_resources = alicloud.Provider("singapore", region="ap-southeast-1")
us_resources = alicloud.Provider("us", region="us-west-1")

# VPC in Singapore
sg_vpc = alicloud.vpc.Network(
    "sg-vpc",
    vpc_name="vpc-sg",
    cidr_block="172.16.0.0/16",
    region_id="ap-southeast-1",
    opts=pulumi.ResourceOptions(provider=singapore_resources),
)

# VPC in US
us_vpc = alicloud.vpc.Network(
    "us-vpc",
    vpc_name="vpc-us",
    cidr_block="172.17.0.0/16",
    region_id="us-west-1",
    opts=pulumi.ResourceOptions(provider=us_resources),
)

# Cross-region replication for critical data
# (Implementation depends on specific service requirements)
```

## Disaster Recovery and High Availability

1. **Multi-Zone Deployment**: Deploy resources across multiple availability zones
2. **Cross-Region Backup**: Implement cross-region backup strategies for critical data
3. **Database Replication**: Use RDS high availability options for database redundancy
4. **Health Checks**: Implement health checks and automated failover mechanisms
5. **Backup Strategies**: Regular automated backups with appropriate retention policies