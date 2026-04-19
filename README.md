# Amazon Fault Injection Simulator (amazon-fault-injection-simulator)

AWS Fault Injection Simulator (FIS) is a fully managed service for running fault injection experiments on AWS. It allows you to improve an application's performance, observability, and resiliency by identifying and fixing weaknesses through controlled chaos engineering experiments.

**URL:** [https://raw.githubusercontent.com/api-evangelist/amazon-fault-injection-simulator/refs/heads/main/apis.yml](https://raw.githubusercontent.com/api-evangelist/amazon-fault-injection-simulator/refs/heads/main/apis.yml)

**Run:** [Capabilities Using Naftiko](https://github.com/naftiko/fleet?utm_source=api-evangelist&utm_medium=readme&utm_campaign=company-api-evangelist&utm_content=repo)

## Tags:

 - AWS, Chaos Engineering, DevOps, Fault Injection, Resilience Testing

## Timestamps

- **Created:** 2026-03-16
- **Modified:** 2026-04-19

## APIs

### AWS Fault Injection Simulator API
The AWS Fault Injection Simulator API provides programmatic access to create and manage experiment templates, experiments, and actions for conducting chaos engineering experiments on AWS workloads.

**Human URL:** [https://aws.amazon.com/fis/](https://aws.amazon.com/fis/)

#### Tags:

 - Chaos Engineering, Fault Injection, Resilience Testing

#### Properties

- [Documentation](https://docs.aws.amazon.com/fis/latest/APIReference/Welcome.html)
- [OpenAPI](openapi/amazon-fis-openapi.yml)
- [JSONSchema](json-schema/amazon-fis-experiment-template-schema.json)
- [JSONSchema](json-schema/amazon-fis-experiment-schema.json)
- [JSONSchema](json-schema/amazon-fis-action-schema.json)
- [JSONSchema](json-schema/amazon-fis-safety-lever-schema.json)
- [JSONStructure](json-structure/amazon-fis-experiment-template-structure.json)
- [JSONStructure](json-structure/amazon-fis-experiment-structure.json)
- [Example](examples/amazon-fis-experiment-template-example.json)
- [Example](examples/amazon-fis-experiment-example.json)
- [GettingStarted](https://aws.amazon.com/fis/getting-started/)
- [Pricing](https://aws.amazon.com/fis/pricing/)
- [FAQ](https://aws.amazon.com/fis/faqs/)
- [APIReference](https://docs.aws.amazon.com/fis/latest/APIReference/Welcome.html)

## Common Properties

- [Portal](https://aws.amazon.com/fis/)
- [Website](https://aws.amazon.com/fis/)
- [Documentation](https://docs.aws.amazon.com/fis/)
- [TermsOfService](https://aws.amazon.com/service-terms/)
- [PrivacyPolicy](https://aws.amazon.com/privacy/)
- [Support](https://aws.amazon.com/premiumsupport/)
- [Blog](https://aws.amazon.com/blogs/devops/tag/aws-fault-injection-simulator/)
- [GitHubOrganization](https://github.com/aws)
- [Console](https://console.aws.amazon.com/fis/)
- [SignUp](https://portal.aws.amazon.com/billing/signup)
- [StatusPage](https://health.aws.amazon.com/health/status)
- [YouTube](https://www.youtube.com/user/AmazonWebServices)
- [StackOverflow](https://stackoverflow.com/questions/tagged/aws-fis)
- [SpectralRules](rules/amazon-fis-spectral-rules.yml)
- [NaftikoCapability](capabilities/shared/fis.yaml)
- [NaftikoCapability](capabilities/amazon-fis-chaos-engineering.yaml)
- [Vocabulary](vocabulary/amazon-fis-vocabulary.yaml)
- [JSON-LD](json-ld/amazon-fis-context.jsonld)

## Features

| Name | Description |
|------|-------------|
| Managed Fault Injection | Fully managed service requiring no agent installation with pre-built fault injection actions for EC2, RDS, ECS, EKS, and more. |
| Pre-built Scenarios | Ready-to-use resilience scenarios for AZ failures, power interruptions, network disruptions, and cross-region connectivity issues. |
| Safety Controls | CloudWatch alarm-based stop conditions and safety levers prevent unintended impact during live testing. |
| Fine-grained Targeting | Tag-based resource targeting scopes experiments to specific environments, applications, or resource subsets. |
| Multi-account Support | Run experiments across multiple AWS accounts using target account configurations. |
| CI/CD Integration | API and CLI access enables automated resilience testing in deployment pipelines. |
| Real-time Visibility | Console and API provide real-time status of executing actions, affected resources, and triggered stop conditions. |
| IAM Security | Fine-grained IAM controls restrict which users can create, run, or view experiments and affected resources. |

## Use Cases

| Name | Description |
|------|-------------|
| Application Resilience Testing | Validate application behavior under resource failures before they occur in production. |
| Chaos Engineering | Run structured fault injection experiments following chaos engineering principles. |
| Observability Validation | Verify that monitoring and alerting systems detect and respond to failures correctly. |
| Game Days | Conduct planned game day exercises simulating failure scenarios for team readiness. |
| Automated Pipeline Testing | Integrate resilience testing into CI/CD pipelines for continuous validation. |
| Multi-region Failover Testing | Test cross-region failover mechanisms and recovery time objectives. |

## Integrations

| Name | Description |
|------|-------------|
| Amazon CloudWatch | Stop conditions use CloudWatch alarms to automatically halt experiments. |
| AWS IAM | Task execution roles define which AWS resources experiments can affect. |
| Amazon EC2 | Stop instances, terminate instances, and inject CPU/memory stress on EC2. |
| Amazon ECS | Stop ECS tasks and inject faults into containerized workloads. |
| Amazon EKS | Terminate Kubernetes nodes and pods running on EKS. |
| Amazon RDS | Trigger RDS failovers, reboot instances, and pause cluster I/O. |
| AWS Lambda | Inject latency and errors into Lambda function invocations. |
| Amazon DynamoDB | Pause DynamoDB replication between replicas. |

## Artifacts

Machine-readable API specifications organized by format.

### OpenAPI

- [amazon-fis-openapi.yml](openapi/amazon-fis-openapi.yml)

### JSON Schema

- [amazon-fis-action-schema.json](json-schema/amazon-fis-action-schema.json)
- [amazon-fis-experiment-schema.json](json-schema/amazon-fis-experiment-schema.json)
- [amazon-fis-experiment-state-schema.json](json-schema/amazon-fis-experiment-state-schema.json)
- [amazon-fis-experiment-template-action-schema.json](json-schema/amazon-fis-experiment-template-action-schema.json)
- [amazon-fis-experiment-template-schema.json](json-schema/amazon-fis-experiment-template-schema.json)
- [amazon-fis-experiment-template-stop-condition-schema.json](json-schema/amazon-fis-experiment-template-stop-condition-schema.json)
- [amazon-fis-experiment-template-target-schema.json](json-schema/amazon-fis-experiment-template-target-schema.json)
- [amazon-fis-safety-lever-schema.json](json-schema/amazon-fis-safety-lever-schema.json)
- [amazon-fis-safety-lever-state-schema.json](json-schema/amazon-fis-safety-lever-state-schema.json)
- [amazon-fis-target-resource-type-schema.json](json-schema/amazon-fis-target-resource-type-schema.json)

### JSON Structure

- [amazon-fis-action-structure.json](json-structure/amazon-fis-action-structure.json)
- [amazon-fis-experiment-state-structure.json](json-structure/amazon-fis-experiment-state-structure.json)
- [amazon-fis-experiment-structure.json](json-structure/amazon-fis-experiment-structure.json)
- [amazon-fis-experiment-template-action-structure.json](json-structure/amazon-fis-experiment-template-action-structure.json)
- [amazon-fis-experiment-template-stop-condition-structure.json](json-structure/amazon-fis-experiment-template-stop-condition-structure.json)
- [amazon-fis-experiment-template-structure.json](json-structure/amazon-fis-experiment-template-structure.json)
- [amazon-fis-experiment-template-target-structure.json](json-structure/amazon-fis-experiment-template-target-structure.json)
- [amazon-fis-safety-lever-state-structure.json](json-structure/amazon-fis-safety-lever-state-structure.json)
- [amazon-fis-safety-lever-structure.json](json-structure/amazon-fis-safety-lever-structure.json)
- [amazon-fis-target-resource-type-structure.json](json-structure/amazon-fis-target-resource-type-structure.json)

### JSON-LD

- [amazon-fis-context.jsonld](json-ld/amazon-fis-context.jsonld)

### Examples

- [amazon-fis-action-example.json](examples/amazon-fis-action-example.json)
- [amazon-fis-experiment-example.json](examples/amazon-fis-experiment-example.json)
- [amazon-fis-experiment-state-example.json](examples/amazon-fis-experiment-state-example.json)
- [amazon-fis-experiment-template-action-example.json](examples/amazon-fis-experiment-template-action-example.json)
- [amazon-fis-experiment-template-example.json](examples/amazon-fis-experiment-template-example.json)
- [amazon-fis-experiment-template-stop-condition-example.json](examples/amazon-fis-experiment-template-stop-condition-example.json)
- [amazon-fis-experiment-template-target-example.json](examples/amazon-fis-experiment-template-target-example.json)
- [amazon-fis-safety-lever-example.json](examples/amazon-fis-safety-lever-example.json)
- [amazon-fis-safety-lever-state-example.json](examples/amazon-fis-safety-lever-state-example.json)
- [amazon-fis-target-resource-type-example.json](examples/amazon-fis-target-resource-type-example.json)

## Capabilities

Naftiko capabilities organized as shared per-API definitions composed into customer-facing workflows.

### Shared Per-API Definitions

- [fis.yaml](capabilities/shared/fis.yaml)

### Workflow Capabilities

| Workflow | APIs Combined | Tools | Persona |
|----------|--------------|-------|---------|
| [amazon-fis-chaos-engineering.yaml](capabilities/amazon-fis-chaos-engineering.yaml) | Amazon Fault Injection Simulator API | — | Platform Engineers, DevOps |

## Vocabulary

- [Amazon Fault Injection Simulator Vocabulary](vocabulary/amazon-fis-vocabulary.yaml)

## Rules

- [amazon-fis-spectral-rules.yml](rules/amazon-fis-spectral-rules.yml)

## Maintainers

**FN:** Kin Lane

**Email:** kin@apievangelist.com
