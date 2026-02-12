# Pulumi Alibaba Cloud Best Practices (Go)

## Provider Configuration

```go
package main

import (
    "github.com/pulumi/pulumi-alicloud/sdk/v3/go/alicloud"
    "github.com/pulumi/pulumi/sdk/v3/go/pulumi"
)

func main() {
    pulumi.Run(func(ctx *pulumi.Context) error {
        // Use ESC for credentials via OIDC

        // Multi-region deployments with explicit providers
        singaporeProvider, err := alicloud.NewProvider(ctx, "singapore", &alicloud.ProviderArgs{
            Region: pulumi.String("ap-southeast-1"),
        })
        if err != nil {
            return err
        }

        usProvider, err := alicloud.NewProvider(ctx, "us", &alicloud.ProviderArgs{
            Region: pulumi.String("us-west-1"),
        })
        if err != nil {
            return err
        }

        // Use provider with resources
        vpc, err := alicloud.NewVpcNetwork(ctx, "vpc", &alicloud.VpcNetworkArgs{
            VpcName:  pulumi.String("my-vpc"),
            CidrBlock: pulumi.String("172.16.0.0/16"),
        }, pulumi.Provider(singaporeProvider))
        if err != nil {
            return err
        }

        return nil
    })
}
```

## Essential Resources

### VPC - Network Pattern

```go
package main

import (
    "github.com/pulumi/pulumi-alicloud/sdk/v3/go/alicloud"
    "github.com/pulumi/pulumi-alicloud/sdk/v3/go/alicloud/vpc"
    "github.com/pulumi/pulumi/sdk/v3/go/pulumi"
)

func main() {
    pulumi.Run(func(ctx *pulumi.Context) error {
        vpc, err := alicloud.NewVpcNetwork(ctx, "vpc", &alicloud.VpcNetworkArgs{
            VpcName:  pulumi.String(ctx.Project()).ToStringOutput().ApplyT(func(project string) string {
                return "vpc-" + project + "-" + ctx.Stack()
            }).(pulumi.StringOutput),
            CidrBlock: pulumi.String("172.16.0.0/16"),
            RegionId:  pulumi.String("ap-southeast-1"), // Using Singapore region
            Description: pulumi.String("VPC for application infrastructure"),
            
            // Tags for resource organization
            Tags: pulumi.StringMap{
                "Environment": pulumi.String(ctx.Stack()),
                "Project":     pulumi.String(ctx.Project()),
                "ManagedBy":   pulumi.String("Pulumi"),
            },
        })
        if err != nil {
            return err
        }

        // Get available zones for the region
        zones, err := alicloud.GetZones(ctx, &alicloud.GetZonesArgs{
            AvailableResourceCreation: pulumi.String("VSwitch"),
            RegionId:                pulumi.String("ap-southeast-1"), // Using Singapore region
        }, nil)
        if err != nil {
            return err
        }

        // Create subnets in multiple zones for high availability
        vswitch, err := alicloud.NewVpcSwitch(ctx, "vswitch", &alicloud.VpcSwitchArgs{
            VswitchName: pulumi.String(ctx.Stack()).ToStringOutput().ApplyT(func(stack string) string {
                return "subnet-" + stack
            }).(pulumi.StringOutput),
            RegionId:  pulumi.String("ap-southeast-1"), // Using Singapore region
            CidrBlock: pulumi.String("172.16.0.0/24"),
            ZoneId:    pulumi.String(zones.Zones[0].Id),
            VpcId:     vpc.ID(),
            
            Tags: pulumi.StringMap{
                "Environment": pulumi.String(ctx.Stack()),
                "Purpose":     pulumi.String("public"),
            },
        })
        if err != nil {
            return err
        }

        // Security group for network access control
        securityGroup, err := alicloud.NewEcsSecurityGroup(ctx, "sg", &alicloud.EcsSecurityGroupArgs{
            SecurityGroupName: pulumi.String(ctx.Stack()).ToStringOutput().ApplyT(func(stack string) string {
                return "sg-" + stack
            }).(pulumi.StringOutput),
            RegionId:    pulumi.String("ap-southeast-1"), // Using Singapore region
            VpcId:       vpc.ID(),
            Description: pulumi.String("Security group for application servers"),
            
            Tags: pulumi.StringMap{
                "Environment": pulumi.String(ctx.Stack()),
            },
        })
        if err != nil {
            return err
        }

        // Add security group rules
        _, err = alicloud.NewEcsSecurityGroupRule(ctx, "ssh-ingress", &alicloud.EcsSecurityGroupRuleArgs{
            Type:       pulumi.String("ingress"),
            RegionId:   pulumi.String("ap-southeast-1"), // Using Singapore region
            IpProtocol: pulumi.String("tcp"),
            NicType:    pulumi.String("intranet"),
            Policy:     pulumi.String("accept"),
            PortRange:  pulumi.String("22/22"),
            Priority:   pulumi.Int(1),
            SecurityGroupId: securityGroup.ID(),
            CidrIp:     pulumi.String("0.0.0.0/0"), // Restrict this in production
        })
        if err != nil {
            return err
        }

        _, err = alicloud.NewEcsSecurityGroupRule(ctx, "http-ingress", &alicloud.EcsSecurityGroupRuleArgs{
            Type:       pulumi.String("ingress"),
            RegionId:   pulumi.String("ap-southeast-1"), // Using Singapore region
            IpProtocol: pulumi.String("tcp"),
            NicType:    pulumi.String("intranet"),
            Policy:     pulumi.String("accept"),
            PortRange:  pulumi.String("80/80"),
            Priority:   pulumi.Int(100),
            SecurityGroupId: securityGroup.ID(),
            CidrIp:     pulumi.String("0.0.0.0/0"),
        })
        if err != nil {
            return err
        }

        ctx.Export("vpcId", vpc.ID())
        ctx.Export("vswitchId", vswitch.ID())
        ctx.Export("securityGroupId", securityGroup.ID())

        return nil
    })
}
```

### ECS - Elastic Compute Service Pattern

```go
package main

import (
    "github.com/pulumi/pulumi-alicloud/sdk/v3/go/alicloud"
    "github.com/pulumi/pulumi/sdk/v3/go/pulumi"
    "github.com/pulumi/pulumi/sdk/v3/go/pulumi/config"
)

func main() {
    pulumi.Run(func(ctx *pulumi.Context) error {
        cfg := config.New(ctx, "")

        isProd := ctx.Stack() == "prod"

        // Create KMS key for disk encryption
        kmsKey, err := alicloud.NewKmsKey(ctx, "encryption-key", &alicloud.KmsKeyArgs{
            RegionId:              pulumi.String("ap-southeast-1"), // Using Singapore region
            Description:           pulumi.String("Key for encrypting ECS data disks"),
            PendingWindowInDays:   pulumi.Int(7),
            Status:                pulumi.String("Enabled"),
            
            Tags: pulumi.StringMap{
                "Environment": pulumi.String(ctx.Stack()),
                "Purpose":     pulumi.String("encryption"),
            },
        })
        if err != nil {
            return err
        }

        // ECS instance with security and encryption
        instance, err := alicloud.NewEcsInstance(ctx, "web-server", &alicloud.EcsInstanceArgs{
            InstanceName: pulumi.String(ctx.Project()).ToStringOutput().ApplyT(func(project string) string {
                return "web-" + project + "-" + ctx.Stack()
            }).(pulumi.StringOutput),
            Description: pulumi.String("Web server instance"),
            
            // Instance configuration
            InstanceType: pulumi.String(func() string {
                if isProd {
                    return "ecs.c6.large"
                }
                return "ecs.t6.small"
            }()),
            ImageId:      pulumi.String("ubuntu_20_04_x64_20G_alibase_20230713.vhd"), // Use latest image
            RegionId:     pulumi.String("ap-southeast-1"), // Using Singapore region
            
            // Network configuration
            VswitchId:      vswitch.ID(),
            SecurityGroups: pulumi.StringArray{securityGroup.ID()},
            
            // Storage configuration
            SystemDiskCategory: pulumi.String("cloud_essd"),
            SystemDiskSize:     pulumi.Int(40),
            
            // Security and encryption
            KeyPairName:      pulumi.String("my-keypair"), // SSH key pair name
            DeletionProtection: pulumi.Bool(isProd),
            
            // Billing
            InstanceChargeType:        pulumi.String("PostPaid"), // More flexible for dev environments
            InternetMaxBandwidthOut: pulumi.Int(func() int {
                if isProd {
                    return 10
                }
                return 5
            }()),
            
            // Tags
            Tags: pulumi.StringMap{
                "Environment": pulumi.String(ctx.Stack()),
                "Role":        pulumi.String("web-server"),
                "ManagedBy":   pulumi.String("Pulumi"),
            },
            
            // Data disks with encryption
            DataDisks: alicloud.EcsInstanceDataDiskArray{
                &alicloud.EcsInstanceDataDiskArgs{
                    Name:        pulumi.String("data-disk"),
                    Size:        pulumi.Int(100),
                    Category:    pulumi.String("cloud_essd"),
                    Encrypted:   pulumi.Bool(true),
                    KmsKeyId:    kmsKey.ID(),
                    DeleteWithInstance: pulumi.Bool(true),
                },
            },
        })
        if err != nil {
            return err
        }

        ctx.Export("instanceId", instance.ID())
        ctx.Export("instancePrivateIp", instance.PrivateIp)
        ctx.Export("instancePublicIp", instance.PublicIp)

        return nil
    })
}
```

### RDS - Database Pattern

```go
package main

import (
    "github.com/pulumi/pulumi-alicloud/sdk/v3/go/alicloud"
    "github.com/pulumi/pulumi/sdk/v3/go/pulumi"
    "github.com/pulumi/pulumi/sdk/v3/go/pulumi/config"
)

func main() {
    pulumi.Run(func(ctx *pulumi.Context) error {
        cfg := config.New(ctx, "")

        isProd := ctx.Stack() == "prod"

        // RDS instance with security and backup
        rdsInstance, err := alicloud.NewRdsInstance(ctx, "postgres-db", &alicloud.RdsInstanceArgs{
            InstanceName: pulumi.String(ctx.Project()).ToStringOutput().ApplyT(func(project string) string {
                return "db-" + project + "-" + ctx.Stack()
            }).(pulumi.StringOutput),
            
            // Engine and version
            Engine:        pulumi.String("PostgreSQL"),
            EngineVersion: pulumi.String("12.0"), // Use latest stable version
            
            // Instance configuration
            InstanceType:    pulumi.String(func() string {
                if isProd {
                    return "rds.pg.x4.large"
                }
                return "rds.pg.x2.small"
            }()),
            InstanceStorage: pulumi.Int(func() int {
                if isProd {
                    return 100
                }
                return 20
            }()), // GB
            RegionId:        pulumi.String("ap-southeast-1"), // Using Singapore region
            
            // Network configuration
            VpcId:      vpc.ID(),
            VswitchId:  vswitch.ID(),
            ZoneId:     pulumi.String(zones.Zones[0].Id),
            
            // Security
            SecurityIps:                pulumi.StringArray{pulumi.String("127.0.0.1")}, // Restrict access in production
            DbInstanceStorageType:      pulumi.String("cloud_essd"), // High performance storage
            EncryptionKey:              kmsKey.ID(), // Use KMS encryption
            
            // Backup and maintenance
            BackupRetentionPeriod: pulumi.Int(7),
            MaintainTime:          pulumi.String("03:00Z-04:00Z"), // Maintenance window
            
            // Billing
            InstanceChargeType: pulumi.String("PostPaid"),
            
            // High availability (production only)
            Category: pulumi.String(func() string {
                if isProd {
                    return "HighAvailability"
                }
                return "Basic"
            }()),
            
            // Security
            DeletionProtection: pulumi.Bool(isProd),
            
            // Tags
            Tags: pulumi.StringMap{
                "Environment": pulumi.String(ctx.Stack()),
                "Tier":        pulumi.String("database"),
                "ManagedBy":   pulumi.String("Pulumi"),
            },
        })
        if err != nil {
            return err
        }

        // Create database within the instance
        database, err := alicloud.NewRdsDatabase(ctx, "app-database", &alicloud.RdsDatabaseArgs{
            InstanceId:   rdsInstance.ID(),
            RegionId:     pulumi.String("ap-southeast-1"), // Using Singapore region
            Name:         pulumi.String("app_db"),
            CharacterSet: pulumi.String("utf8mb4"),
        })
        if err != nil {
            return err
        }

        // Create database account
        account, err := alicloud.NewRdsAccount(ctx, "app-account", &alicloud.RdsAccountArgs{
            InstanceId:     rdsInstance.ID(),
            RegionId:       pulumi.String("ap-southeast-1"), // Using Singapore region
            AccountName:    pulumi.String("app_user"),
            AccountPassword: cfg.RequireSecret("db_password"),
            AccountType:    pulumi.String("Normal"),
        })
        if err != nil {
            return err
        }

        ctx.Export("rdsConnectionString", rdsInstance.ConnectionString)
        ctx.Export("rdsPort", rdsInstance.Port)

        return nil
    })
}
```

### Object Storage Service (OSS) - Secure Bucket Pattern

```go
package main

import (
    "github.com/pulumi/pulumi-alicloud/sdk/v3/go/alicloud"
    "github.com/pulumi/pulumi/sdk/v3/go/pulumi"
)

func main() {
    pulumi.Run(func(ctx *pulumi.Context) error {
        ossBucket, err := alicloud.NewOssBucket(ctx, "data-bucket", &alicloud.OssBucketArgs{
            Bucket:   pulumi.String(ctx.Project()).ToStringOutput().ApplyT(func(project string) string {
                return "data-" + project + "-" + ctx.Stack()
            }).(pulumi.StringOutput),
            RegionId: pulumi.String("ap-southeast-1"), // Using Singapore region
            
            // Storage class for cost optimization
            StorageClass: pulumi.String("Standard"),
            
            // Server-side encryption
            ServerSideEncryptionRule: &alicloud.OssBucketServerSideEncryptionRuleArgs{
                SseAlgorithm:   pulumi.String("KMS"),
                KmsMasterKeyId: kmsKey.ID(),
            },
            
            // Versioning for data protection
            Versioning: pulumi.String("Enabled"),
            
            // Access control
            Acl: pulumi.String("private"),
            
            // Lifecycle rules for cost optimization
            LifecycleRules: alicloud.OssBucketLifecycleRuleArray{
                &alicloud.OssBucketLifecycleRuleArgs{
                    Id:     pulumi.String("transition-to-ia"),
                    Status: pulumi.String("Enabled"),
                    Prefix: pulumi.String("logs/"),
                    Actions: alicloud.OssBucketLifecycleRuleActionArray{
                        &alicloud.OssBucketLifecycleRuleActionArgs{
                            Type:                   pulumi.String("Transition"),
                            StorageClass:           pulumi.String("IA"),
                            TransitionDelayedDays: pulumi.Int(30),
                        },
                        &alicloud.OssBucketLifecycleRuleActionArgs{
                            Type:                   pulumi.String("Transition"),
                            StorageClass:           pulumi.String("Archive"),
                            TransitionDelayedDays: pulumi.Int(90),
                        },
                        &alicloud.OssBucketLifecycleRuleActionArgs{
                            Type:                   pulumi.String("Expiration"),
                            ExpirationDelayedDays: pulumi.Int(365),
                        },
                    },
                },
            },
            
            // Tags
            Tags: pulumi.StringMap{
                "Environment": pulumi.String(ctx.Stack()),
                "Purpose":     pulumi.String("data-storage"),
                "ManagedBy":   pulumi.String("Pulumi"),
            },
        })
        if err != nil {
            return err
        }

        // Bucket policy to deny insecure transport
        bucketPolicy, err := alicloud.NewOssBucketPolicy(ctx, "secure-policy", &alicloud.OssBucketPolicyArgs{
            Bucket:   ossBucket.ID(),
            RegionId: pulumi.String("ap-southeast-1"), // Using Singapore region
            Policy: ossBucket.ID().ApplyT(func(id string) string {
                return `{{
                    "Version": "1",
                    "Statement": [
                        {{
                            "Effect": "Deny",
                            "Principal": "*",
                            "Action": "oss:GetObject",
                            "Resource": "acs:oss:*:*:` + id + `/*",
                            "Condition": {{
                                "Bool": {{
                                    "acs:SecureTransport": "false"
                                }}
                            }}
                        }}
                    ]
                }}`
            }).(pulumi.StringOutput),
        })
        if err != nil {
            return err
        }

        ctx.Export("ossBucketName", ossBucket.Bucket)
        ctx.Export("ossBucketDomain", ossBucket.ExtranetEndpoint)

        return nil
    })
}
```

### Application Load Balancer (ALB) Pattern

```go
package main

import (
    "github.com/pulumi/pulumi-alicloud/sdk/v3/go/alicloud"
    "github.com/pulumi/pulumi/sdk/v3/go/pulumi"
)

func main() {
    pulumi.Run(func(ctx *pulumi.Context) error {
        // Application Load Balancer
        alb, err := alicloud.NewAlbLoadBalancer(ctx, "application-lb", &alicloud.AlbLoadBalancerArgs{
            LoadBalancerName: pulumi.String(ctx.Project()).ToStringOutput().ApplyT(func(project string) string {
                return "alb-" + project + "-" + ctx.Stack()
            }).(pulumi.StringOutput),
            AddressType: pulumi.String("Internet"),
            RegionId:    pulumi.String("ap-southeast-1"), // Using Singapore region
            VpcId:       vpc.ID(),
            VswitchIds:  pulumi.StringArray{vswitch.ID()},

            // Load balancer specification
            LoadBalancerEdition: pulumi.String("Standard"),
            LoadBalancerBillingConfig: &alicloud.AlbLoadBalancerLoadBalancerBillingConfigArgs{
                PayType: pulumi.String("PayAsYouYou"),
            },

            // Security
            DeletionProtectionEnabled: pulumi.Bool(true),

            // Tags
            Tags: pulumi.StringMap{
                "Environment": pulumi.String(ctx.Stack()),
                "Purpose":     pulumi.String("load-balancer"),
                "ManagedBy":   pulumi.String("Pulumi"),
            },
        })
        if err != nil {
            return err
        }

        // Server group for backend instances
        serverGroup, err := alicloud.NewAlbServerGroup(ctx, "backend-servers", &alicloud.AlbServerGroupArgs{
            ServerGroupName: pulumi.String(ctx.Stack()).ToStringOutput().ApplyT(func(stack string) string {
                return "backend-" + stack
            }).(pulumi.StringOutput),
            RegionId:       pulumi.String("ap-southeast-1"), // Using Singapore region
            LoadBalancerId: alb.ID(),

            // Backend servers
            Servers: alicloud.AlbServerGroupServerArray{
                &alicloud.AlbServerGroupServerArgs{
                    ServerId:   instance.ID(),
                    ServerType: pulumi.String("Ecs"),
                    Weight:     pulumi.Int(100),
                    Port:       pulumi.Int(80),
                },
            },
        })
        if err != nil {
            return err
        }

        // Listener for HTTP traffic
        listener, err := alicloud.NewAlbListener(ctx, "http-listener", &alicloud.AlbListenerArgs{
            ListenerName:   pulumi.String(ctx.Stack()).ToStringOutput().ApplyT(func(stack string) string {
                return "http-" + stack
            }).(pulumi.StringOutput),
            RegionId:       pulumi.String("ap-southeast-1"), // Using Singapore region
            LoadBalancerId: alb.ID(),
            Protocol:       pulumi.String("HTTP"),
            Port:           pulumi.Int(80),

            // Default actions
            DefaultActions: alicloud.AlbListenerDefaultActionArray{
                &alicloud.AlbListenerDefaultActionArgs{
                    Type:          pulumi.String("ForwardGroup"),
                    ForwardGroupId: serverGroup.ID(),
                },
            },
        })
        if err != nil {
            return err
        }

        ctx.Export("albDnsName", alb.DnsName)
        ctx.Export("albId", alb.ID())

        return nil
    })
}
```

## Security Best Practices

### RAM - Resource Access Management

```go
package main

import (
    "github.com/pulumi/pulumi-alicloud/sdk/v3/go/alicloud"
    "github.com/pulumi/pulumi/sdk/v3/go/pulumi"
)

func main() {
    pulumi.Run(func(ctx *pulumi.Context) error {
        // Create a custom policy for minimal permissions
        customPolicy, err := alicloud.NewRamPolicy(ctx, "app-policy", &alicloud.RamPolicyArgs{
            PolicyName: pulumi.String(ctx.Project()).ToStringOutput().ApplyT(func(project string) string {
                return "policy-" + project + "-" + ctx.Stack()
            }).(pulumi.StringOutput),
            RegionId:    pulumi.String("ap-southeast-1"), // Using Singapore region
            Description: pulumi.String("Custom policy for application access"),
            PolicyDocument: ossBucket.ID().ApplyT(func(bucketId string) string {
                return `{{
                    "Version": "1",
                    "Statement": [
                        {{
                            "Effect": "Allow",
                            "Action": [
                                "oss:GetObject",
                                "oss:PutObject"
                            ],
                            "Resource": [
                                "acs:oss:*:*:` + bucketId + `/*"
                            ]
                        }}
                    ]
                }}`
            }).(pulumi.StringOutput),
            Type: pulumi.String("Custom"),
        })
        if err != nil {
            return err
        }

        // Create a RAM role for ECS instances
        role, err := alicloud.NewRamRole(ctx, "ecs-role", &alicloud.RamRoleArgs{
            RoleName: pulumi.String(ctx.Project()).ToStringOutput().ApplyT(func(project string) string {
                return "role-" + project + "-" + ctx.Stack()
            }).(pulumi.StringOutput),
            RegionId: pulumi.String("ap-southeast-1"), // Using Singapore region
            Document: pulumi.String(`{
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
            }`),
            Description: pulumi.String("Role for ECS instances"),
        })
        if err != nil {
            return err
        }

        // Attach policy to role
        _, err = alicloud.NewRamAttachment(ctx, "attach-policy", &alicloud.RamAttachmentArgs{
            RoleName:     role.Name,
            RegionId:     pulumi.String("ap-southeast-1"), // Using Singapore region
            PolicyName:   customPolicy.Name,
            PolicyType:   customPolicy.Type,
        })
        if err != nil {
            return err
        }

        return nil
    })
}
```

### KMS - Key Management Service

```go
package main

import (
    "github.com/pulumi/pulumi-alicloud/sdk/v3/go/alicloud"
    "github.com/pulumi/pulumi/sdk/v3/go/pulumi"
)

func main() {
    pulumi.Run(func(ctx *pulumi.Context) error {
        // Already shown in previous examples, but important for security
        kmsKey, err := alicloud.NewKmsKey(ctx, "encryption-key", &alicloud.KmsKeyArgs{
            RegionId:              pulumi.String("ap-southeast-1"), // Using Singapore region
            Description:           pulumi.String("Key for encrypting sensitive data"),
            PendingWindowInDays:   pulumi.Int(7),
            Status:                pulumi.String("Enabled"),
            
            Tags: pulumi.StringMap{
                "Environment": pulumi.String(ctx.Stack()),
                "Purpose":     pulumi.String("encryption"),
            },
        })
        if err != nil {
            return err
        }

        return nil
    })
}
```

## Auto-Scaling

```go
package main

import (
    "github.com/pulumi/pulumi-alicloud/sdk/v3/go/alicloud"
    "github.com/pulumi/pulumi/sdk/v3/go/pulumi"
)

func main() {
    pulumi.Run(func(ctx *pulumi.Context) error {
        // Auto Scaling Group
        scalingGroup, err := alicloud.NewEssScalingGroup(ctx, "web-scaling-group", &alicloud.EssScalingGroupArgs{
            ScalingGroupName: pulumi.String(ctx.Project()).ToStringOutput().ApplyT(func(project string) string {
                return "scaling-" + project + "-" + ctx.Stack()
            }).(pulumi.StringOutput),
            MaxSize:  pulumi.Int(10),
            MinSize:  pulumi.Int(2),
            RegionId: pulumi.String("ap-southeast-1"), // Using Singapore region
            VswitchIds: pulumi.StringArray{vswitch.ID()},
            RemovalPolicies: pulumi.StringArray{
                pulumi.String("OldestInstance"),
                pulumi.String("NewestInstance"),
            },
            
            Tags: pulumi.StringMap{
                "Environment": pulumi.String(ctx.Stack()),
                "Purpose":     pulumi.String("auto-scaling"),
            },
        })
        if err != nil {
            return err
        }

        // Scaling configuration
        scalingConfig, err := alicloud.NewEssScalingConfiguration(ctx, "web-config", &alicloud.EssScalingConfigurationArgs{
            ScalingGroupId: scalingGroup.ID(),
            RegionId:       pulumi.String("ap-southeast-1"), // Using Singapore region
            ImageId:        pulumi.String("ubuntu_20_04_x64_20G_alibase_20230713.vhd"),
            InstanceType:   pulumi.String("ecs.c6.large"),
            SecurityGroupId: securityGroup.ID(),
            KeyPairName:    pulumi.String("my-keypair"),

            Tags: pulumi.StringMap{
                "Environment": pulumi.String(ctx.Stack()),
            },
        })
        if err != nil {
            return err
        }

        // Scaling rule
        _, err = alicloud.NewEssScalingRule(ctx, "cpu-scale-out", &alicloud.EssScalingRuleArgs{
            ScalingGroupId: scalingGroup.ID(),
            RegionId:       pulumi.String("ap-southeast-1"), // Using Singapore region
            AdjustmentType: pulumi.String("QuantityChangeInCapacity"),
            AdjustmentValue: pulumi.Int(2),
            ScalingRuleName: pulumi.String("scale-out-rule"),
            Cooldown:        pulumi.Int(300),
        })
        if err != nil {
            return err
        }

        return nil
    })
}
```

## Tagging Strategy

For Go, tags are typically applied directly to resources as Go doesn't have the same transformation capabilities as other languages. You can define a helper function to standardize tags:

```go
package main

import (
    "github.com/pulumi/pulumi/sdk/v3/go/pulumi"
)

func getDefaultTags(ctx *pulumi.Context) pulumi.StringMap {
    return pulumi.StringMap{
        "Environment":  pulumi.String(ctx.Stack()),
        "Project":      pulumi.String(ctx.Project()),
        "ManagedBy":    pulumi.String("Pulumi"),
        "Team":         pulumi.String("engineering"),     // Customize based on your organization
        "BusinessUnit": pulumi.String("web-services"),   // Customize based on your organization
    }
}

// Then use it in resource creation:
// tags := getDefaultTags(ctx)
// vpc, err := alicloud.NewVpcNetwork(ctx, "vpc", &alicloud.VpcNetworkArgs{
//     // ... other args
//     Tags: tags,
// })
```

## Multi-Region Deployment

For global applications, consider deploying resources across multiple regions:

```go
package main

import (
    "github.com/pulumi/pulumi-alicloud/sdk/v3/go/alicloud"
    "github.com/pulumi/pulumi/sdk/v3/go/pulumi"
)

func main() {
    pulumi.Run(func(ctx *pulumi.Context) error {
        // Create resources in multiple regions
        singaporeResources, err := alicloud.NewProvider(ctx, "singapore", &alicloud.ProviderArgs{
            Region: pulumi.String("ap-southeast-1"),
        })
        if err != nil {
            return err
        }

        usResources, err := alicloud.NewProvider(ctx, "us", &alicloud.ProviderArgs{
            Region: pulumi.String("us-west-1"),
        })
        if err != nil {
            return err
        }

        // VPC in Singapore
        sgVpc, err := alicloud.NewVpcNetwork(ctx, "sg-vpc", &alicloud.VpcNetworkArgs{
            VpcName:   pulumi.String("vpc-sg"),
            CidrBlock: pulumi.String("172.16.0.0/16"),
            RegionId:  pulumi.String("ap-southeast-1"),
        }, pulumi.Provider(singaporeResources))
        if err != nil {
            return err
        }

        // VPC in US
        usVpc, err := alicloud.NewVpcNetwork(ctx, "us-vpc", &alicloud.VpcNetworkArgs{
            VpcName:   pulumi.String("vpc-us"),
            CidrBlock: pulumi.String("172.17.0.0/16"),
            RegionId:  pulumi.String("us-west-1"),
        }, pulumi.Provider(usResources))
        if err != nil {
            return err
        }

        // Cross-region replication for critical data
        // (Implementation depends on specific service requirements)

        return nil
    })
}
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