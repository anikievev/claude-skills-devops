---
name: aws-cdk
description: Write well-structured AWS CDK stacks following construct separation, encryption, least-privilege IAM, and environment-agnostic design best practices
---

Analyze the current project and write production-grade AWS CDK code. If CDK code already exists, review and improve it.

## Requirements

**Inspect the project first:**
- Detect language (TypeScript, Python, Java, Go)
- Check for existing `cdk.json`, `cdk.context.json`, stack files
- Identify what AWS resources are being provisioned

**Project structure — use this layout (TypeScript example):**

```
.
├── bin/
│   └── app.ts          # CDK App entry point — instantiates stacks
├── lib/
│   ├── constructs/     # Reusable L3 constructs (one file per logical group)
│   └── stacks/         # Stack definitions (one file per stack)
├── config/
│   └── config.ts       # Environment-specific config (loaded from context)
├── cdk.json
└── cdk.context.json    # Resolved context values (commit this file)
```

**Mandatory rules:**

### App entry point (`bin/app.ts`)
- Always pass explicit `env: { account, region }` using `process.env.CDK_DEFAULT_ACCOUNT` and `CDK_DEFAULT_REGION` — never hardcode account/region
- Instantiate one stack per logical boundary (networking, compute, data, etc.)
- Pass typed config objects to stacks — never read env vars inside constructs

### Stack design
- One stack per deployment unit — keep stacks small and independently deployable
- Set `RemovalPolicy` explicitly on every stateful resource:
  - Production: `RemovalPolicy.RETAIN`
  - Dev/test: `RemovalPolicy.DESTROY` — but only if explicitly configured via context
- Add stack-level tags using `Tags.of(this).add(key, value)` for all resources
- Use `CfnOutput` for values needed by other stacks or CI/CD (ARNs, URLs, names)
- Cross-stack references via `Fn.importValue` / `CfnOutput` — never pass stack object references across independent stacks

### Constructs
- Create L3 constructs for any resource group that appears more than once
- Constructs must accept a typed `Props` interface — no `any`, no loose `Record<string,string>`
- Keep AWS service logic inside constructs; keep orchestration in stacks

### IAM — least privilege
- Use `grant*` methods (`grantRead`, `grantWrite`, `grantInvoke`) over raw `addToPolicy`
- When raw policies are needed, scope to minimum actions and specific ARN resources — never `Resource: "*"` unless justified with a comment
- Use `PrincipalWithConditions` for cross-account access; always add `aws:RequestedRegion` conditions

### Encryption
- Enable encryption at rest for all resources that support it:
  - S3: `encryption: BucketEncryption.S3_MANAGED` (or KMS for compliance requirements)
  - RDS/Aurora: `storageEncrypted: true` with a managed KMS key
  - DynamoDB: `encryption: TableEncryption.AWS_MANAGED`
  - SQS/SNS: use KMS keys
- Enable encryption in transit: enforce HTTPS on S3 (`enforceSSL: true`), use TLS for RDS

### CDK Aspects
- Apply a tagging Aspect at the App level to enforce mandatory tags on all resources
- Apply a compliance Aspect to enforce encryption, versioning, or other org-wide policies

### Context (`cdk.context.json` / `cdk.json`)
- Store environment-specific values (VPC IDs, account numbers, domain names) in context — not hardcoded
- Access via `this.node.tryGetContext('key')` with a fallback or a required-or-throw pattern

## Output format

Provide all files in full with fenced code blocks labeled by filename:

```typescript
// lib/stacks/MyStack.ts
...
```

After the code, include:
1. A **Deploy** section with the exact CDK commands: `cdk synth`, `cdk diff`, `cdk deploy`
2. A **Security notes** section flagging IAM or encryption trade-offs
3. Flag any resource with `RemovalPolicy.DESTROY` and state the condition under which it applies
