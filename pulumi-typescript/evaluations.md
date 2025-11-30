# Evaluation Scenarios for pulumi-typescript

## Scenario 1: Create S3 Bucket with ESC Integration

**Input:** "Create a Pulumi TypeScript program to deploy an encrypted S3 bucket, and show me how to set up ESC for the configuration"

**Expected Behavior:**

- Activate when "Pulumi" and "TypeScript" are mentioned
- Generate TypeScript code with proper imports
- Create S3 bucket with encryption and versioning
- Show ESC environment setup for configuration
- Use `pulumi.Config()` to read from ESC
- Add proper tags using `pulumi.getStack()`

**Success Criteria:**

- [ ] TypeScript code generated (index.ts)
- [ ] Imports @pulumi/aws and @pulumi/pulumi
- [ ] Bucket has serverSideEncryptionConfiguration
- [ ] Shows ESC environment YAML with pulumiConfig block
- [ ] Demonstrates `esc env init` and `pulumi config env add`
- [ ] Tags include Environment and ManagedBy

## Scenario 2: Multi-Stack with ESC OIDC

**Input:** "Set up a Pulumi TypeScript project with OIDC authentication to AWS using ESC, with separate dev and prod environments"

**Expected Behavior:**

- Show ESC environment with aws-login OIDC provider
- Create environment composition (base + dev/prod)
- Demonstrate stack references if needed
- Show `esc run` for deployment

**Success Criteria:**

- [ ] ESC YAML with fn::open::aws-login
- [ ] OIDC roleArn configuration shown
- [ ] Environment variables exported (AWS_ACCESS_KEY_ID, etc.)
- [ ] Shows how to link ESC to stack
- [ ] Demonstrates esc run command

## Scenario 3: Component Resource Pattern

**Input:** "Create a reusable VPC component in Pulumi TypeScript that I can use across projects"

**Expected Behavior:**

- Create ComponentResource class
- Proper typing for inputs
- Use { parent: this } for child resources
- Register outputs
- Show usage example

**Success Criteria:**

- [ ] Extends pulumi.ComponentResource
- [ ] Constructor with typed args
- [ ] super() call with custom type URN
- [ ] Child resources use { parent: this }
- [ ] registerOutputs called

## Scenario 4: Dynamic Secrets from AWS Secrets Manager

**Input:** "How do I pull secrets from AWS Secrets Manager into my Pulumi TypeScript project using ESC?"

**Expected Behavior:**

- Show ESC environment with aws-secrets provider
- Chain OIDC login to secrets access
- Expose secrets via pulumiConfig
- Show TypeScript code accessing secrets

**Success Criteria:**

- [ ] ESC YAML with fn::open::aws-secrets
- [ ] Uses ${aws.login} for authentication
- [ ] Secrets exposed in pulumiConfig block
- [ ] TypeScript uses config.requireSecret()
- [ ] Warns about not logging secrets
