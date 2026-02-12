# Pulumi Alibaba Cloud Best Practices (TypeScript)

## Provider Configuration

```typescript
import * as pulumi from "@pulumi/pulumi";
import * as alicloud from "@pulumi/alicloud";

// Use ESC for credentials via OIDC

// Multi-region deployments with explicit providers
const singaporeProvider = new alicloud.Provider("singapore", { region: "ap-southeast-1" });
const usProvider = new alicloud.Provider("us", { region: "us-west-1" });

// Use provider with resources
const vpc = new alicloud.vpc.Network("vpc", {
    vpcName: "my-vpc",
    cidrBlock: "172.16.0.0/16",
}, { provider: singaporeProvider });
```

## Essential Resources

### VPC - Network Pattern

```typescript
import * as pulumi from "@pulumi/pulumi";
import * as alicloud from "@pulumi/alicloud";

const vpc = new alicloud.vpc.Network("vpc", {
    vpcName: `vpc-${pulumi.getProject()}-${pulumi.getStack()}`,
    cidrBlock: "172.16.0.0/16",
    regionId: "ap-southeast-1",  // Using Singapore region
    description: "VPC for application infrastructure",
    
    // Tags for resource organization
    tags: {
        Environment: pulumi.getStack(),
        Project: pulumi.getProject(),
        ManagedBy: "Pulumi",
    },
});

// Get available zones for the region
const zones = alicloud.getZones({
    availableResourceCreation: "VSwitch",
    regionId: "ap-southeast-1",  // Using Singapore region
});

// Create subnets in multiple zones for high availability
const vswitch = new alicloud.vpc.Switch("vswitch", {
    vswitchName: `subnet-${pulumi.getStack()}`,
    regionId: "ap-southeast-1",  // Using Singapore region
    cidrBlock: "172.16.0.0/24",
    zoneId: zones.then(z => z.zones?.[0]?.id),
    vpcId: vpc.id,
    
    tags: {
        Environment: pulumi.getStack(),
        Purpose: "public",
    },
});

// Security group for network access control
const securityGroup = new alicloud.ecs.SecurityGroup("sg", {
    securityGroupName: `sg-${pulumi.getStack()}`,
    regionId: "ap-southeast-1",  // Using Singapore region
    vpcId: vpc.id,
    description: "Security group for application servers",
    
    tags: {
        Environment: pulumi.getStack(),
    },
});

// Add security group rules
const sshRule = new alicloud.ecs.SecurityGroupRule("ssh-ingress", {
    type: "ingress",
    regionId: "ap-southeast-1",  // Using Singapore region
    ipProtocol: "tcp",
    nicType: "intranet",
    policy: "accept",
    portRange: "22/22",
    priority: 1,
    securityGroupId: securityGroup.id,
    cidrIp: "0.0.0.0/0",  // Restrict this in production
});

const httpRule = new alicloud.ecs.SecurityGroupRule("http-ingress", {
    type: "ingress",
    regionId: "ap-southeast-1",  // Using Singapore region
    ipProtocol: "tcp",
    nicType: "intranet",
    policy: "accept",
    portRange: "80/80",
    priority: 100,
    securityGroupId: securityGroup.id,
    cidrIp: "0.0.0.0/0",
});

export const vpcId = vpc.id;
export const vswitchId = vswitch.id;
export const securityGroupId = securityGroup.id;
```

### ECS - Elastic Compute Service Pattern

```typescript
import * as pulumi from "@pulumi/pulumi";
import * as alicloud from "@pulumi/alicloud";

const config = new pulumi.Config();
const isProd = pulumi.getStack() === "prod";

// Create KMS key for disk encryption
const kmsKey = new alicloud.kms.Key("encryption-key", {
    regionId: "ap-southeast-1",  // Using Singapore region
    description: "Key for encrypting ECS data disks",
    pendingWindowInDays: 7,
    status: "Enabled",
    
    tags: {
        Environment: pulumi.getStack(),
        Purpose: "encryption",
    },
});

// ECS instance with security and encryption
const instance = new alicloud.ecs.Instance("web-server", {
    instanceName: `web-${pulumi.getProject()}-${pulumi.getStack()}`,
    description: "Web server instance",
    
    // Instance configuration
    instanceType: isProd ? "ecs.c6.large" : "ecs.t6.small",
    imageId: "ubuntu_20_04_x64_20G_alibase_20230713.vhd",  // Use latest image
    regionId: "ap-southeast-1",  // Using Singapore region
    
    // Network configuration
    vswitchId: vswitch.id,
    securityGroups: [securityGroup.id],
    
    // Storage configuration
    systemDiskCategory: "cloud_essd",
    systemDiskSize: 40,
    
    // Security and encryption
    keyPairName: "my-keypair",  // SSH key pair name
    deletionProtection: isProd,
    
    // Billing
    instanceChargeType: "PostPaid",  // More flexible for dev environments
    internetMaxBandwidthOut: isProd ? 10 : 5,
    
    // Tags
    tags: {
        Environment: pulumi.getStack(),
        Role: "web-server",
        ManagedBy: "Pulumi",
    },
    
    // Data disks with encryption
    dataDisks: [{
        name: "data-disk",
        size: 100,
        category: "cloud_essd",
        encrypted: true,
        kmsKeyId: kmsKey.id,
        deleteWithInstance: true,
    }],
});

export const instanceId = instance.id;
export const instancePrivateIp = instance.privateIp;
export const instancePublicIp = instance.publicIp;
```

### RDS - Database Pattern

```typescript
import * as pulumi from "@pulumi/pulumi";
import * as alicloud from "@pulumi/alicloud";

const config = new pulumi.Config();
const isProd = pulumi.getStack() === "prod";

// RDS instance with security and backup
const rdsInstance = new alicloud.rds.Instance("postgres-db", {
    instanceName: `db-${pulumi.getProject()}-${pulumi.getStack()}`,
    
    // Engine and version
    engine: "PostgreSQL",
    engineVersion: "12.0",  // Use latest stable version
    
    // Instance configuration
    instanceType: isProd ? "rds.pg.x4.large" : "rds.pg.x2.small",
    instanceStorage: isProd ? 100 : 20,  // GB
    
    // Network configuration
    vpcId: vpc.id,
    vswitchId: vswitch.id,
    zoneId: zones.then(z => z.zones?.[0]?.id),
    regionId: "ap-southeast-1",  // Using Singapore region
    
    // Security
    securityIps: ["127.0.0.1"],  // Restrict access in production
    dbInstanceStorageType: "cloud_essd",  // High performance storage
    encryptionKey: kmsKey.id,  // Use KMS encryption
    
    // Backup and maintenance
    backupRetentionPeriod: 7,
    maintainTime: "03:00Z-04:00Z",  // Maintenance window
    
    // Billing
    instanceChargeType: "PostPaid",
    
    // High availability (production only)
    category: isProd ? "HighAvailability" : "Basic",
    
    // Security
    deletionProtection: isProd,
    
    // Tags
    tags: {
        Environment: pulumi.getStack(),
        Tier: "database",
        ManagedBy: "Pulumi",
    },
});

// Create database within the instance
const database = new alicloud.rds.Database("app-database", {
    instanceId: rdsInstance.id,
    regionId: "ap-southeast-1",  // Using Singapore region
    name: "app_db",
    characterSet: "utf8mb4",
});

// Create database account
const account = new alicloud.rds.Account("app-account", {
    instanceId: rdsInstance.id,
    regionId: "ap-southeast-1",  // Using Singapore region
    accountName: "app_user",
    accountPassword: config.requireSecret("db_password"),
    accountType: "Normal",
});

export const rdsConnectionString = rdsInstance.connectionString;
export const rdsPort = rdsInstance.port;
```

### Object Storage Service (OSS) - Secure Bucket Pattern

```typescript
import * as pulumi from "@pulumi/pulumi";
import * as alicloud from "@pulumi/alicloud";

const ossBucket = new alicloud.oss.Bucket("data-bucket", {
    bucket: `data-${pulumi.getProject()}-${pulumi.getStack()}`,
    regionId: "ap-southeast-1",  // Using Singapore region
    
    // Storage class for cost optimization
    storageClass: "Standard",
    
    // Server-side encryption
    serverSideEncryptionRule: {
        sseAlgorithm: "KMS",
        kmsMasterKeyId: kmsKey.id,
    },
    
    // Versioning for data protection
    versioning: "Enabled",
    
    // Access control
    acl: "private",
    
    // Lifecycle rules for cost optimization
    lifecycleRule: [
        {
            id: "transition-to-ia",
            status: "Enabled",
            prefix: "logs/",
            actions: [
                {
                    type: "Transition",
                    storageClass: "IA",
                    transitionDelayedDays: 30,
                },
                {
                    type: "Transition",
                    storageClass: "Archive",
                    transitionDelayedDays: 90,
                },
                {
                    type: "Expiration",
                    expirationDelayedDays: 365,
                },
            ],
        },
    ],
    
    // Tags
    tags: {
        Environment: pulumi.getStack(),
        Purpose: "data-storage",
        ManagedBy: "Pulumi",
    },
});

// Bucket policy to deny insecure transport
const bucketPolicy = new alicloud.oss.BucketPolicy("secure-policy", {
    bucket: ossBucket.id,
    regionId: "ap-southeast-1",  // Using Singapore region
    policy: pulumi.interpolate`{
        "Version": "1",
        "Statement": [
            {
                "Effect": "Deny",
                "Principal": "*",
                "Action": "oss:GetObject",
                "Resource": "acs:oss:*:*:${ossBucket.id}/*",
                "Condition": {
                    "Bool": {
                        "acs:SecureTransport": "false"
                    }
                }
            }
        ]
    }`,
});

export const ossBucketName = ossBucket.bucket;
export const ossBucketDomain = ossBucket.extranetEndpoint;
```

### Application Load Balancer (ALB) Pattern

```typescript
import * as pulumi from "@pulumi/pulumi";
import * as alicloud from "@pulumi/alicloud";

// Application Load Balancer
const alb = new alicloud.alb.LoadBalancer("application-lb", {
    loadBalancerName: `alb-${pulumi.getProject()}-${pulumi.getStack()}`,
    addressType: "Internet",
    regionId: "ap-southeast-1",  // Using Singapore region
    vpcId: vpc.id,
    vswitchIds: [vswitch.id],

    // Load balancer specification
    loadBalancerEdition: "Standard",
    loadBalancerBillingConfig: {
        payType: "PayAsYouYou",
    },

    // Security
    deletionProtectionEnabled: true,

    // Tags
    tags: {
        Environment: pulumi.getStack(),
        Purpose: "load-balancer",
        ManagedBy: "Pulumi",
    },
});

// Listener for HTTP traffic
const listener = new alicloud.alb.Listener("http-listener", {
    listenerName: `http-${pulumi.getStack()}`,
    regionId: "ap-southeast-1",  // Using Singapore region
    loadBalancerId: alb.id,
    protocol: "HTTP",
    port: 80,

    // Default actions
    defaultActions: [{
        type: "ForwardGroup",
        forwardGroupId: serverGroup.id,
    }],
});

// Server group for backend instances
const serverGroup = new alicloud.alb.ServerGroup("backend-servers", {
    serverGroupName: `backend-${pulumi.getStack()}`,
    regionId: "ap-southeast-1",  // Using Singapore region
    loadBalancerId: alb.id,

    // Backend servers
    servers: [{
        serverId: instance.id,
        serverType: "Ecs",
        weight: 100,
        port: 80,
    }],
});

export const albDnsName = alb.dnsName;
export const albId = alb.id;
```

## Security Best Practices

### RAM - Resource Access Management

```typescript
import * as pulumi from "@pulumi/pulumi";
import * as alicloud from "@pulumi/alicloud";

// Create a custom policy for minimal permissions
const customPolicy = new alicloud.ram.Policy("app-policy", {
    policyName: `policy-${pulumi.getProject()}-${pulumi.getStack()}`,
    regionId: "ap-southeast-1",  // Using Singapore region
    description: "Custom policy for application access",
    policyDocument: pulumi.interpolate`{
        "Version": "1",
        "Statement": [
            {
                "Effect": "Allow",
                "Action": [
                    "oss:GetObject",
                    "oss:PutObject"
                ],
                "Resource": [
                    "acs:oss:*:*:${ossBucket.id}/*"
                ]
            }
        ]
    }`,
    type: "Custom",
});

// Create a RAM role for ECS instances
const role = new alicloud.ram.Role("ecs-role", {
    roleName: `role-${pulumi.getProject()}-${pulumi.getStack()}`,
    regionId: "ap-southeast-1",  // Using Singapore region
    document: `{
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
    }`,
    description: "Role for ECS instances",
});

// Attach policy to role
const policyAttachment = new alicloud.ram.Attachment("attach-policy", {
    roleName: role.name,
    regionId: "ap-southeast-1",  // Using Singapore region
    policyName: customPolicy.name,
    policyType: customPolicy.type,
});
```

### KMS - Key Management Service

```typescript
import * as alicloud from "@pulumi/alicloud";

// Already shown in previous examples, but important for security
const kmsKey = new alicloud.kms.Key("encryption-key", {
    regionId: "ap-southeast-1",  // Using Singapore region
    description: "Key for encrypting sensitive data",
    pendingWindowInDays: 7,
    status: "Enabled",
    
    tags: {
        Environment: pulumi.getStack(),
        Purpose: "encryption",
    },
});
```

## Auto-Scaling

```typescript
import * as alicloud from "@pulumi/alicloud";

// Auto Scaling Group
const scalingGroup = new alicloud.ess.ScalingGroup("web-scaling-group", {
    scalingGroupName: `scaling-${pulumi.getProject()}-${pulumi.getStack()}`,
    maxSize: 10,
    minSize: 2,
    regionId: "ap-southeast-1",  // Using Singapore region
    vswitchIds: [vswitch.id],
    removalPolicies: ["OldestInstance", "NewestInstance"],
    
    tags: {
        Environment: pulumi.getStack(),
        Purpose: "auto-scaling",
    },
});

// Scaling configuration
const scalingConfig = new alicloud.ess.ScalingConfiguration("web-config", {
    scalingGroupId: scalingGroup.id,
    regionId: "ap-southeast-1",  // Using Singapore region
    imageId: "ubuntu_20_04_x64_20G_alibase_20230713.vhd",
    instanceType: "ecs.c6.large",
    securityGroupId: securityGroup.id,
    keyPairName: "my-keypair",

    tags: {
        Environment: pulumi.getStack(),
    },
});

// Scaling rule
const scalingRule = new alicloud.ess.ScalingRule("cpu-scale-out", {
    scalingGroupId: scalingGroup.id,
    regionId: "ap-southeast-1",  // Using Singapore region
    adjustmentType: "QuantityChangeInCapacity",
    adjustmentValue: 2,
    scalingRuleName: "scale-out-rule",
    cooldown: 300,
});
```

## Tagging Strategy

```typescript
import * as pulumi from "@pulumi/pulumi";

// Standard tags for all Alibaba Cloud resources
const defaultTags = {
    Environment: pulumi.getStack(),
    Project: pulumi.getProject(),
    ManagedBy: "Pulumi",
    Team: "engineering",  // Customize based on your organization
    BusinessUnit: "web-services",  // Customize based on your organization
};

// Register transformation to add tags to all resources
pulumi.runtime.registerStackTransformation((args) => {
    if (args.type.startsWith("alicloud:") && args.props.tags !== undefined) {
        // Merge default tags with existing tags, preferring existing tags
        const mergedTags = { ...defaultTags, ...args.props.tags };
        return { props: { ...args.props, tags: mergedTags }, opts: args.opts };
    } else if (args.type.startsWith("alicloud:")) {
        // If tags property exists but is undefined, set it to default tags
        return { props: { ...args.props, tags: defaultTags }, opts: args.opts };
    }
    return { props: args.props, opts: args.opts };
});
```

## Multi-Region Deployment

For global applications, consider deploying resources across multiple regions:

```typescript
import * as pulumi from "@pulumi/pulumi";
import * as alicloud from "@pulumi/alicloud";

// Create resources in multiple regions
const singaporeResources = new alicloud.Provider("singapore", { region: "ap-southeast-1" });
const usResources = new alicloud.Provider("us", { region: "us-west-1" });

// VPC in Singapore
const sgVpc = new alicloud.vpc.Network("sg-vpc", {
    vpcName: "vpc-sg",
    cidrBlock: "172.16.0.0/16",
    regionId: "ap-southeast-1",
}, { provider: singaporeResources });

// VPC in US
const usVpc = new alicloud.vpc.Network("us-vpc", {
    vpcName: "vpc-us",
    cidrBlock: "172.17.0.0/16",
    regionId: "us-west-1",
}, { provider: usResources });

// Cross-region replication for critical data
// (Implementation depends on specific service requirements)
```

## Cost Optimization Tips

1. **Use Reserved Instances**: For predictable workloads, consider reserved instances for significant cost savings
2. **Right-size Resources**: Regularly review and adjust instance sizes based on actual usage
3. **Use Spot Instances**: For fault-tolerant workloads, spot instances can provide up to 90% discount
4. **Implement Lifecycle Policies**: Use OSS lifecycle rules to transition data to lower-cost storage classes
5. **Monitor Resource Utilization**: Use CloudMonitor to identify underutilized resources
6. **Clean Up Unused Resources**: Regularly remove unused snapshots, images, and other resources
7. **Use Auto-Scaling**: Implement auto-scaling to match capacity with demand

## Disaster Recovery and High Availability

1. **Multi-Zone Deployment**: Deploy resources across multiple availability zones
2. **Cross-Region Backup**: Implement cross-region backup strategies for critical data
3. **Database Replication**: Use RDS high availability options for database redundancy
4. **Health Checks**: Implement health checks and automated failover mechanisms
5. **Backup Strategies**: Regular automated backups with appropriate retention policies