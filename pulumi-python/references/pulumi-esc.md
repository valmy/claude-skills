# Pulumi ESC Reference

## Overview

Pulumi ESC (Environments, Secrets, and Configuration) provides centralized secrets management and orchestration for infrastructure and applications.

## ESC CLI Commands

### Environment Management

```bash
# Create environment
esc env init <org>/<project>/<env>
esc env init myorg/myproject/dev

# List environments
esc env ls
esc env ls myorg

# Edit environment (opens editor)
esc env edit <org>/<env>

# View environment definition
esc env get <org>/<env> --show-secrets

# Delete environment
esc env rm <org>/<env>

# Clone environment
esc env clone <org>/<source> <org>/<target>
```

### Working with Values

```bash
# Set a value
esc env set <org>/<env> <path> <value>
esc env set myorg/dev pulumiConfig.aws:region us-west-2

# Set a secret
esc env set <org>/<env> <path> <value> --secret

# Get a specific value
esc env get <org>/<env> <path>
esc env get myorg/dev pulumiConfig.aws:region
```

### Running Commands

```bash
# Run command with environment
esc run <org>/<env> -- <command>
esc run myorg/aws-dev -- pulumi up
esc run myorg/aws-dev -- aws s3 ls

# Open environment (resolve and display)
esc open <org>/<env>
esc open <org>/<env> --format json
esc open <org>/<env> --format shell
esc open <org>/<env> --format dotenv
```

### Versioning

```bash
# List versions
esc env version ls <org>/<env>

# Tag a version
esc env version tag <org>/<env> <tag>
esc env version tag myorg/prod stable

# Rollback to version
esc env version rollback <org>/<env> <version>

# Diff versions
esc env diff <org>/<env>@<v1> <org>/<env>@<v2>
```

## ESC Environment Syntax

### Basic Structure

```yaml
imports:
  - base-environment

values:
  staticConfig:
    region: us-west-2
    environment: production

  secrets:
    apiKey:
      fn::secret: "my-secret-value"

  pulumiConfig:
    aws:region: ${staticConfig.region}
    myapp:apiKey: ${secrets.apiKey}

  environmentVariables:
    AWS_REGION: ${staticConfig.region}
    MY_API_KEY: ${secrets.apiKey}
```

### Dynamic Providers

#### AWS OIDC Login

```yaml
values:
  aws:
    login:
      fn::open::aws-login:
        oidc:
          roleArn: arn:aws:iam::123456789012:role/pulumi-oidc-role
          sessionName: pulumi-${context.pulumi.user.login}
          duration: 1h
```

#### AWS Secrets Manager

```yaml
values:
  secrets:
    fn::open::aws-secrets:
      region: us-west-2
      login: ${aws.login}
      get:
        dbPassword:
          secretId: prod/db/password
        apiKey:
          secretId: prod/api/key
```

#### Azure OIDC Login

```yaml
values:
  azure:
    login:
      fn::open::azure-login:
        clientId: <app-client-id>
        tenantId: <tenant-id>
        subscriptionId: <subscription-id>
        oidc: true
```

#### GCP OIDC Login

```yaml
values:
  gcp:
    login:
      fn::open::gcp-login:
        project: my-project-id
        oidc:
          workloadPoolId: pulumi-pool
          providerId: pulumi-provider
          serviceAccount: pulumi@my-project.iam.gserviceaccount.com
```

## Integration with Pulumi IaC

### Linking ESC to Stacks

```bash
pulumi config env add <org>/<env>
pulumi config env rm <org>/<env>
pulumi config env ls
```

### Accessing Config in Python Code

```python
import pulumi

config = pulumi.Config()
aws_config = pulumi.Config("aws")

# Values from pulumiConfig block
region = aws_config.require("region")
instance_type = config.require("myapp:instanceType")

# Secrets (returns Output[str])
api_key = config.require_secret("myapp:apiKey")

# Optional values with defaults
debug = config.get_bool("debug") or False
```
