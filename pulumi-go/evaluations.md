# Evaluation Scenarios for pulumi-go

## Scenario 1: Create S3 Bucket with ESC Integration

**Input:** "Create a Pulumi Go program to deploy an encrypted S3 bucket, and show me how to set up ESC for the configuration"

**Expected Behavior:**

- Activate when "Pulumi" and "Go" are mentioned
- Generate Go code with proper imports and error handling
- Create S3 bucket with encryption and versioning
- Show ESC environment setup for configuration
- Use `config.New()` to read from ESC
- Add proper tags using `ctx.Stack()`

**Success Criteria:**

- [ ] Go code generated (main.go)
- [ ] Imports github.com/pulumi/pulumi-aws/sdk and github.com/pulumi/pulumi/sdk
- [ ] Bucket has ServerSideEncryptionConfiguration
- [ ] Shows ESC environment YAML with pulumiConfig block
- [ ] Demonstrates `esc env init` and `pulumi config env add`
- [ ] Tags include Environment and ManagedBy
- [ ] All errors are checked and wrapped

## Scenario 2: Component Resource in Go

**Input:** "Create a reusable VPC component in Pulumi Go with public and private subnets"

**Expected Behavior:**

- Create struct implementing component pattern
- Use RegisterComponentResource
- Define Args struct with proper input types
- Use pulumi.Parent(component) for child resources
- Register outputs properly

**Success Criteria:**

- [ ] VpcArgs struct defined with pulumi.Input types
- [ ] Vpc struct with pulumi.ResourceState embedded
- [ ] Output fields with pulumi tags
- [ ] RegisterComponentResource called
- [ ] Child resources use pulumi.Parent(component)
- [ ] RegisterResourceOutputs called

## Scenario 3: Working with Outputs

**Input:** "Show me how to transform and combine outputs in Pulumi Go"

**Expected Behavior:**

- Demonstrate ApplyT for transformations
- Show pulumi.All for combining outputs
- Proper type assertions
- Error handling in transformations

**Success Criteria:**

- [ ] ApplyT with proper type assertion
- [ ] pulumi.All for multiple outputs
- [ ] Type assertions to correct output types
- [ ] Example of error handling in ApplyT

## Scenario 4: ESC with OIDC for AWS

**Input:** "Set up Pulumi ESC with OIDC authentication for AWS in a Go project"

**Expected Behavior:**

- Show ESC environment with aws-login
- Configure OIDC role
- Export environment variables
- Link to Pulumi stack
- Show esc run usage

**Success Criteria:**

- [ ] ESC YAML with fn::open::aws-login
- [ ] OIDC roleArn and sessionName configured
- [ ] AWS credentials in environmentVariables
- [ ] Commands for linking ESC to stack
- [ ] esc run example for deployment
