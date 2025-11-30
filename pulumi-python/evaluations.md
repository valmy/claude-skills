# Evaluation Scenarios for pulumi-python

## Scenario 1: Create S3 Bucket with ESC Integration

**Input:** "Create a Pulumi Python program to deploy an encrypted S3 bucket, and show me how to set up ESC for the configuration"

**Expected Behavior:**

- Activate when "Pulumi" and "Python" are mentioned
- Generate Python code with proper imports
- Create S3 bucket with encryption and versioning
- Show ESC environment setup for configuration
- Use `pulumi.Config()` to read from ESC
- Add proper tags using `pulumi.get_stack()`

**Success Criteria:**

- [ ] Python code generated (__main__.py)
- [ ] Imports pulumi and pulumi_aws
- [ ] Bucket has server_side_encryption_configuration
- [ ] Shows ESC environment YAML with pulumiConfig block
- [ ] Demonstrates `esc env init` and `pulumi config env add`
- [ ] Tags include Environment and ManagedBy
- [ ] Uses either Args classes or dict literals consistently

## Scenario 2: Component Resource in Python

**Input:** "Create a reusable VPC component in Pulumi Python with type hints"

**Expected Behavior:**

- Create class extending ComponentResource
- Define Args class with type hints
- Use opts=pulumi.ResourceOptions(parent=self) for child resources
- Register outputs properly

**Success Criteria:**

- [ ] VpcArgs class with type hints
- [ ] Vpc class extends pulumi.ComponentResource
- [ ] super().__init__ with custom type URN
- [ ] Child resources use parent=self
- [ ] register_outputs called
- [ ] Output type hints on class attributes

## Scenario 3: Working with Outputs

**Input:** "Show me how to transform and combine outputs in Pulumi Python"

**Expected Behavior:**

- Demonstrate apply for transformations
- Show Output.all for combining outputs
- Show Output.concat for string building
- Show Output.format for formatting

**Success Criteria:**

- [ ] .apply() with lambda function
- [ ] pulumi.Output.all() for multiple outputs
- [ ] pulumi.Output.concat() example
- [ ] pulumi.Output.format() example

## Scenario 4: ESC with OIDC for AWS

**Input:** "Set up Pulumi ESC with OIDC authentication for AWS in a Python project"

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

## Scenario 5: Args Classes vs Dict Literals

**Input:** "What's the difference between using Args classes and dict literals in Pulumi Python?"

**Expected Behavior:**

- Explain both approaches
- Show equivalent examples
- Discuss type checking benefits
- Recommend when to use each

**Success Criteria:**

- [ ] Example using BucketVersioningArgs
- [ ] Equivalent example using {"enabled": True}
- [ ] Mentions type checking with Args classes
- [ ] Notes that dicts are more concise
