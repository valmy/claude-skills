# Pulumi Infrastructure Patterns (Go)

## Component Resources

```go
package main

import (
    "github.com/pulumi/pulumi-aws/sdk/v6/go/aws/ec2"
    "github.com/pulumi/pulumi/sdk/v3/go/pulumi"
)

type VpcArgs struct {
    CidrBlock        pulumi.StringInput
    AzCount          int
    EnableNatGateway bool
}

type Vpc struct {
    pulumi.ResourceState

    VpcId            pulumi.StringOutput   `pulumi:"vpcId"`
    PublicSubnetIds  pulumi.StringArrayOutput `pulumi:"publicSubnetIds"`
    PrivateSubnetIds pulumi.StringArrayOutput `pulumi:"privateSubnetIds"`
}

func NewVpc(ctx *pulumi.Context, name string, args *VpcArgs, opts ...pulumi.ResourceOption) (*Vpc, error) {
    component := &Vpc{}
    err := ctx.RegisterComponentResource("custom:network:Vpc", name, component, opts...)
    if err != nil {
        return nil, err
    }

    vpc, err := ec2.NewVpc(ctx, name+"-vpc", &ec2.VpcArgs{
        CidrBlock:          args.CidrBlock,
        EnableDnsHostnames: pulumi.Bool(true),
        EnableDnsSupport:   pulumi.Bool(true),
        Tags: pulumi.StringMap{
            "Name": pulumi.String(name),
        },
    }, pulumi.Parent(component))
    if err != nil {
        return nil, err
    }

    component.VpcId = vpc.ID().ToStringOutput()

    ctx.RegisterResourceOutputs(component, pulumi.Map{
        "vpcId":            component.VpcId,
        "publicSubnetIds":  component.PublicSubnetIds,
        "privateSubnetIds": component.PrivateSubnetIds,
    })

    return component, nil
}
```

## Stack References

```go
func main() {
    pulumi.Run(func(ctx *pulumi.Context) error {
        // Reference networking stack
        networkStack, err := pulumi.NewStackReference(ctx, "myorg/networking/prod", nil)
        if err != nil {
            return err
        }

        vpcId := networkStack.GetStringOutput(pulumi.String("vpcId"))

        // Type assertion for arrays
        subnetIds := networkStack.GetOutput(pulumi.String("privateSubnetIds")).ApplyT(
            func(v interface{}) []string {
                ids := v.([]interface{})
                result := make([]string, len(ids))
                for i, id := range ids {
                    result[i] = id.(string)
                }
                return result
            },
        ).(pulumi.StringArrayOutput)

        return nil
    })
}
```

## Transformations

```go
func main() {
    pulumi.Run(func(ctx *pulumi.Context) error {
        // Register transformation for all resources
        ctx.RegisterStackTransformation(func(args *pulumi.ResourceTransformationArgs) *pulumi.ResourceTransformationResult {
            // Add tags to all taggable resources
            if args.Props != nil {
                if tags, ok := args.Props["tags"]; ok {
                    tagMap := tags.(pulumi.Map)
                    tagMap["Environment"] = pulumi.String(ctx.Stack())
                    tagMap["ManagedBy"] = pulumi.String("Pulumi")
                    args.Props["tags"] = tagMap
                }
            }
            return &pulumi.ResourceTransformationResult{
                Props: args.Props,
                Opts:  args.Opts,
            }
        })

        return nil
    })
}
```

## Configuration Patterns

```go
func main() {
    pulumi.Run(func(ctx *pulumi.Context) error {
        cfg := config.New(ctx, "myapp")

        // Environment-aware defaults
        isProd := ctx.Stack() == "prod"

        instanceType := cfg.Get("instanceType")
        if instanceType == "" {
            if isProd {
                instanceType = "t3.large"
            } else {
                instanceType = "t3.small"
            }
        }

        // Structured configuration
        type DbConfig struct {
            Host string `json:"host"`
            Port int    `json:"port"`
            Name string `json:"name"`
        }

        var dbConfig DbConfig
        cfg.RequireObject("database", &dbConfig)

        return nil
    })
}
```

## Testing Patterns

### Unit Testing with Mocks

```go
package main

import (
    "testing"

    "github.com/pulumi/pulumi/sdk/v3/go/pulumi"
    "github.com/stretchr/testify/assert"
)

type mocks int

func (mocks) NewResource(args pulumi.MockResourceArgs) (string, resource.PropertyMap, error) {
    return args.Name + "_id", args.Inputs, nil
}

func (mocks) Call(args pulumi.MockCallArgs) (resource.PropertyMap, error) {
    return args.Args, nil
}

func TestInfrastructure(t *testing.T) {
    err := pulumi.RunErr(func(ctx *pulumi.Context) error {
        // Test your infrastructure
        return nil
    }, pulumi.WithMocks("project", "stack", mocks(0)))

    assert.NoError(t, err)
}
```

### Integration Testing with Automation API

```go
package main

import (
    "context"
    "testing"

    "github.com/pulumi/pulumi/sdk/v3/go/auto"
    "github.com/stretchr/testify/assert"
)

func TestDeploy(t *testing.T) {
    ctx := context.Background()

    stack, err := auto.UpsertStackLocalSource(ctx, "test", ".")
    assert.NoError(t, err)

    // Deploy
    result, err := stack.Up(ctx)
    assert.NoError(t, err)
    assert.Equal(t, "succeeded", result.Summary.Result)

    // Cleanup
    _, err = stack.Destroy(ctx)
    assert.NoError(t, err)
}
```

## Resource Options

```go
// Common resource options
resource, err := s3.NewBucket(ctx, "bucket", &s3.BucketArgs{},
    // Explicit dependencies
    pulumi.DependsOn([]pulumi.Resource{otherResource}),

    // Parent for components
    pulumi.Parent(component),

    // Protect from deletion
    pulumi.Protect(true),

    // Ignore changes
    pulumi.IgnoreChanges([]string{"tags"}),

    // Custom provider
    pulumi.Provider(customProvider),

    // Aliases for refactoring
    pulumi.Aliases([]pulumi.Alias{{Name: pulumi.String("old-name")}}),

    // Delete before replace
    pulumi.DeleteBeforeReplace(true),
)
```
