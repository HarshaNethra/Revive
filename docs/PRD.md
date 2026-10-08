# Self-Healing AWS Infrastructure Platform

## Product Requirements Document

**Project Type:** Cloud Infrastructure / DevOps / SRE / Backend  
**Primary Cloud:** AWS  
**Supported Resources:** EC2, Lambda, S3  
**Primary Database:** Amazon DynamoDB  
**Primary Goal:** Build a generalized self-healing infrastructure platform capable of detecting, diagnosing, remediating, and verifying failures across multiple AWS services.

---

# 1. Project Overview

## 1.1 Problem Statement

Cloud infrastructure can fail or enter unhealthy states without completely becoming unavailable.

Examples include:

- EC2 applications or processes becoming unresponsive.
- EC2 memory or disk utilization reaching critical levels.
- Lambda functions experiencing abnormal error rates.
- S3 buckets violating defined security or configuration policies.

Traditional operations require engineers to manually detect the issue, investigate it, perform recovery, and verify the result.

This increases recovery time and operational overhead.

The goal of this project is to build a **generalized Self-Healing Infrastructure Platform** that automatically detects predefined failures, diagnoses them, performs approved remediation actions, verifies recovery, and records the complete incident lifecycle.

The initial platform will support:

```text
EC2
Lambda
S3
```

The platform will eventually be deployed as an AWS-native application using multiple AWS services.

---

# 2. Core Engineering Principle

The system follows:

```text
Detect
  ↓
Identify
  ↓
Diagnose
  ↓
Select Policy
  ↓
Remediate
  ↓
Verify
  ↓
Record
  ↓
Notify
```

The architecture separates:

### Core Healing Engine

Generic logic responsible for:

- Event processing.
- Policy selection.
- Incident management.
- Recovery orchestration.
- Verification.
- State management.

### Service Adapters

Service-specific logic responsible for:

- Diagnosis.
- Remediation.
- Verification.

Initial adapters:

```text
EC2Adapter
LambdaAdapter
S3Adapter
```

---

# 3. Project Development Strategy

The project will be developed in three major phases.

```text
PHASE 1
Build Self-Healing Engine
        ↓
PHASE 2
Build Core Application + Control Plane
        ↓
PHASE 3
AWS Deployment
```

AWS deployment is intentionally the **final implementation phase**.

The application and healing engine should be functional before the complete AWS infrastructure is deployed.

---

# 4. Phase 1 — Self-Healing Engine

## 4.1 Objective

Build the core healing engine independently of the final AWS deployment architecture.

The engine should implement the fundamental workflow:

```text
Event
 ↓
Normalize
 ↓
Identify Resource
 ↓
Select Policy
 ↓
Diagnose
 ↓
Remediate
 ↓
Verify
 ↓
Record Result
```

The initial implementation should prioritize correct architecture and recovery logic rather than cloud deployment.

---

# 5. Healing Engine Architecture

```text
┌─────────────────────────────────────┐
│          SELF-HEALING ENGINE        │
├─────────────────────────────────────┤
│                                     │
│ Event Processor                      │
│        ↓                            │
│ Resource Resolver                    │
│        ↓                            │
│ Policy Engine                        │
│        ↓                            │
│ Diagnosis Engine                     │
│        ↓                            │
│ Remediation Engine                   │
│        ↓                            │
│ Verification Engine                  │
│        ↓                            │
│ Incident Manager                     │
│                                     │
└─────────────────────────────────────┘
```

The engine must not contain hardcoded EC2/Lambda/S3 logic in the core workflow.

---

# 6. Service Adapter Architecture

Each supported AWS service will implement a common adapter interface.

Conceptually:

```text
ServiceAdapter

├── diagnose()
├── remediate()
└── verify()
```

Architecture:

```text
                    Healing Engine
                          │
          ┌───────────────┼───────────────┐
          ↓               ↓               ↓
     EC2 Adapter     Lambda Adapter    S3 Adapter
          │               │               │
       EC2 APIs        Lambda APIs       S3 APIs
```

This allows additional services to be added later without modifying the core healing workflow.

---

# 7. Phase 1 — EC2 Adapter

The EC2 adapter should support:

### Diagnosis

- Memory utilization.
- Disk utilization.
- Process state.
- Application health.
- Relevant system information.

### Remediation

Level 1:

```text
Restart Application
```

Level 2:

```text
Safe Temporary-File Cleanup
```

Level 3:

```text
EC2 Reboot
```

### Verification

- Process running.
- Application health endpoint responding.
- Resource utilization returning to acceptable levels.

---

# 8. Phase 1 — Lambda Adapter

The Lambda adapter should support:

### Diagnosis

- Error rate.
- Invocation behavior.
- Duration.
- Current function version.
- Current alias configuration.
- Known-good version availability.

### Remediation

The initial strategy should use a known-good deployment state.

Example:

```text
Current Version
      ↓
Abnormal Errors
      ↓
Known-Good Version Available?
      ↓
Update Alias / Rollback
```

The system must not modify Lambda source code automatically.

### Verification

- Expected version active.
- Subsequent invocation succeeds.
- Error condition recovers.

---

# 9. Phase 1 — S3 Adapter

S3 self-healing will focus on configuration and security compliance rather than server-style recovery.

### Diagnosis

Check:

- Public access configuration.
- Encryption configuration.
- Required bucket policy.
- Required lifecycle/configuration settings.

### Remediation

Restore explicitly defined expected configuration.

### Verification

Re-check the bucket configuration after remediation.

The system must not automatically delete objects.

---

# 10. Healing Policy Engine

Healing behavior must be represented as explicit policies.

Example:

```text
Policy
├── policyId
├── serviceType
├── failureType
├── severity
├── diagnosisAction
├── remediationAction
├── verificationAction
├── maxAttempts
├── cooldown
└── enabled
```

Example:

```text
EC2_PROCESS_FAILURE

Diagnosis:
Process Health Check

Remediation:
Restart Application

Verification:
Application Health Check

Max Attempts:
2
```

The policy engine determines what actions the platform is allowed to perform.

---

# 11. Recovery Levels

The platform should prefer the least disruptive recovery action.

```text
Level 1
Application Recovery
        ↓
Level 2
Resource Cleanup / Secondary Recovery
        ↓
Level 3
Infrastructure Recovery
        ↓
Level 4
Human Escalation
```

Not every service needs all four levels.

For example:

```text
EC2
→ Restart Process
→ Cleanup
→ Reboot
→ Escalate
```

while:

```text
S3
→ Restore Configuration
→ Verify
→ Escalate
```

---

# 12. Recovery Verification

Every remediation action must have a corresponding verification step.

The platform must never assume:

```text
API call succeeded
=
System recovered
```

Instead:

```text
Remediation
     ↓
Verification
     ↓
 ┌───┴────┐
Success  Failure
   ↓        ↓
Resolved  Escalate
```

---

# 13. Idempotency

The engine must prevent duplicate healing operations.

Example:

```text
Event A ──┐
          ├──→ Same Incident
Event B ──┘
```

Only one healing workflow should own the incident.

The system must support:

- Incident ownership.
- Recovery attempt limits.
- State validation.
- Cooldowns.
- Idempotent remediation operations.

---

# 14. Incident State Machine

The healing engine should use an explicit incident state machine.

```text
DETECTED
   ↓
DIAGNOSING
   ↓
REMEDIATING
   ↓
VERIFYING
   │
   ├──────────────→ RECOVERED
   │
   └──────────────→ ESCALATED
                         ↓
                       FAILED
```

Possible states:

```text
DETECTED
DIAGNOSING
REMEDIATING
VERIFYING
RECOVERED
ESCALATED
FAILED
```

---

# 15. Phase 1 — Engine Testing

Before AWS deployment, the healing engine should be tested using simulated events and service adapters.

Example:

```text
Simulated EC2 Process Failure
          ↓
Healing Engine
          ↓
EC2 Adapter
          ↓
Mock Remediation
          ↓
Mock Verification
          ↓
Incident Resolved
```

Tests should cover:

- Successful recovery.
- Failed recovery.
- Multiple recovery attempts.
- Invalid policies.
- Duplicate events.
- Concurrent incidents.
- Verification failure.
- Escalation.

The engine should be considered structurally sound before proceeding to the application/control-plane phase.

---

# 16. Phase 2 — Core Application

## 16.1 Objective

Build the actual application surrounding the healing engine.

The application will provide:

- Resource management.
- Healing policy management.
- Incident management.
- Recovery history.
- System status.
- API endpoints.
- Authentication.
- Operations dashboard.

---

# 17. Control Plane

Amazon DynamoDB will eventually act as the persistent control-plane datastore.

The control plane will store:

```text
Resources
Policies
Incidents
Recovery Attempts
System State
```

Conceptually:

```text
                 Control Plane
                       │
                ┌──────┴──────┐
                ↓             ↓
             DynamoDB       Engine
                │             │
                ├── Resources │
                ├── Policies  │
                ├── Incidents │
                └── Attempts  │
```

---

# 18. DynamoDB Data Model

A DynamoDB single-table design may be used.

Example logical structure:

```text
PK                  SK
──────────────────────────────────────────────
RESOURCE#ec2-123    METADATA
RESOURCE#ec2-123    INCIDENT#2026-0001

RESOURCE#lambda-X   METADATA
RESOURCE#lambda-X   INCIDENT#2026-0002

RESOURCE#bucket-X   METADATA
RESOURCE#bucket-X   INCIDENT#2026-0003

INCIDENT#2026-0001  METADATA
INCIDENT#2026-0001  ATTEMPT#1
INCIDENT#2026-0001  ATTEMPT#2

POLICY#EC2_PROCESS_FAILURE
POLICY#LAMBDA_ERROR_RATE
POLICY#S3_SECURITY_CONFIG
```

The final physical schema must be based on actual application access patterns.

---

# 19. Core Data Models

## Resource

```text
resourceId
serviceType
region
resourceArn
status
monitoringEnabled
activeIncidentId
metadata
createdAt
updatedAt
```

## HealingPolicy

```text
policyId
serviceType
failureType
severity
diagnosisAction
remediationAction
verificationAction
maxAttempts
cooldownSeconds
enabled
createdAt
updatedAt
```

## Incident

```text
incidentId
resourceId
serviceType
failureType
severity
status
detectionSource
detectedAt
diagnosis
selectedPolicy
currentRecoveryLevel
resolvedAt
updatedAt
```

## RecoveryAttempt

```text
incidentId
attemptNumber
action
recoveryLevel
startedAt
completedAt
result
diagnosticOutput
error
```

---

# 20. DynamoDB Concurrency Control

DynamoDB conditional writes should protect incident state transitions.

Example:

```text
Incident Status:
DETECTED

Worker A
   ↓
Conditional Update
DETECTED → DIAGNOSING
   ↓
SUCCESS

Worker B
   ↓
Conditional Update
DETECTED → DIAGNOSING
   ↓
FAIL
```

Worker B must not continue with remediation.

This protects the platform against duplicate events and concurrent healing workflows.

---

# 21. Backend API

The application should expose APIs for the operations dashboard and administrative workflows.

Example endpoints:

```text
GET    /api/resources
GET    /api/resources/{id}

GET    /api/incidents
GET    /api/incidents/{id}

GET    /api/policies
POST   /api/policies
PUT    /api/policies/{id}

GET    /api/recovery-attempts

GET    /api/health
GET    /api/metrics
```

Manual remediation endpoints may be added later, but automatic remediation remains policy-controlled.

---

# 22. Operations Dashboard

The platform should provide an operations dashboard displaying:

```text
Resources
├── EC2
├── Lambda
└── S3

Incidents
├── Active
├── Recovered
├── Escalated
└── Failed

Recovery
├── Attempts
├── Success Rate
└── MTTR
```

Example:

```text
┌─────────────────────────────────────────┐
│      SELF-HEALING CONTROL CENTER        │
├─────────────────────────────────────────┤
│                                         │
│ EC2       ● Healthy                     │
│ Lambda    ● Healthy                     │
│ S3        ● Healthy                     │
│                                         │
│ Active Incidents: 1                     │
│ Recoveries Today: 17                    │
│ Success Rate: 94%                       │
│ Average MTTR: 38 sec                    │
│                                         │
├─────────────────────────────────────────┤
│ Recent Incidents                         │
│                                         │
│ EC2-001  Process Failure    RECOVERED   │
│ LMB-004  Error Rate         RECOVERED   │
│ S3-003   Policy Violation   REMEDIATED  │
└─────────────────────────────────────────┘
```

---

# 23. Authentication and Authorization

The application should eventually use Amazon Cognito.

Required concepts:

```text
User
 ↓
Cognito
 ↓
Authentication
 ↓
Authorization
 ↓
Operations Dashboard
```

Possible roles:

```text
ADMIN
OPERATOR
VIEWER
```

Example permissions:

### ADMIN

- Manage policies.
- Register resources.
- Configure system settings.

### OPERATOR

- View incidents.
- View recovery history.
- Trigger approved manual recovery where supported.

### VIEWER

- Read-only access.

---

# 24. Phase 2 — Local / Development Environment

Before complete AWS deployment, the core application should be runnable in a controlled development environment.

The development environment should support:

```text
Frontend
   ↓
Backend API
   ↓
Healing Engine
   ↓
Development Database / DynamoDB-compatible environment
```

The objective is to validate:

- API behavior.
- Engine integration.
- Data models.
- Incident lifecycle.
- Policy management.
- Dashboard functionality.

AWS-specific deployment infrastructure should be introduced after the application architecture is stable.

---

# 25. Phase 3 — AWS Deployment

## 25.1 Objective

Deploy the complete application and infrastructure onto AWS.

The deployment phase should be the **final major phase**.

At this point:

```text
Engine
   +
Core Application
   +
Database Model
   +
Dashboard
   +
Service Adapters
```

should already be functional.

The deployment phase transforms the working system into a production-style AWS architecture.

---

# 26. AWS Deployment Architecture

The final deployment may use:

```text
                         USERS
                           │
                           ▼
                      CloudFront
                           │
                           ▼
                     S3 Frontend
                           │
                           ▼
                      API Gateway
                           │
                           ▼
                       Cognito
                           │
                           ▼
                  Backend / Lambda
                           │
             ┌─────────────┼─────────────┐
             ▼             ▼             ▼
         DynamoDB         S3        EventBridge
             │                           │
             │                           ▼
             │                    Healing Engine
             │                           │
             │              ┌────────────┼────────────┐
             │              ▼            ▼            ▼
             │             EC2        Lambda         S3
             │              │            │            │
             │              ▼            ▼            ▼
             │             SSM       Lambda API     S3 API
             │
             └──────────────────────────────┐
                                            ▼
                                       CloudWatch
                                            │
                                            ▼
                                        CloudTrail
                                            │
                                            ▼
                                      SNS / SES
```

The exact architecture should be finalized during deployment based on the requirements discovered during implementation.

---

# 27. AWS Services and Responsibilities

## Application Layer

### Amazon S3

- Host frontend static assets.
- Store system artifacts where required.

### Amazon CloudFront

- Distribute the frontend.
- Provide edge delivery.

### API Gateway

- Expose backend APIs.
- Provide API-level controls.

### Amazon Cognito

- User authentication.
- Authorization.

---

## Compute

### AWS Lambda

Used for:

- Backend API functions where appropriate.
- Healing controller.
- Event processing.
- Supporting automation.

### Amazon EC2

Used as:

- A monitored/healable infrastructure target.
- A deliberately failure-prone workload for demonstrating recovery.

---

## Data

### Amazon DynamoDB

Used for:

- Resources.
- Policies.
- Incidents.
- Recovery attempts.
- Platform state.

### Amazon S3

Used for:

- Frontend hosting.
- Object storage.
- S3 self-healing demonstration.

---

## Eventing

### Amazon EventBridge

Used for:

- CloudWatch alarm events.
- AWS service events.
- Triggering healing workflows.

### Amazon SQS

May be introduced to buffer healing events and provide asynchronous processing where required.

---

## Workflow

### AWS Step Functions

May be used for complex recovery workflows such as:

```text
Detect
 ↓
Diagnose
 ↓
Remediate
 ↓
Wait
 ↓
Verify
 ├── Success → Complete
 └── Failure → Retry / Escalate
```

Step Functions should only be introduced where workflow orchestration provides a clear advantage over direct Lambda orchestration.

---

## Monitoring

### Amazon CloudWatch

Used for:

- Metrics.
- Logs.
- Alarms.
- Health monitoring.

### CloudWatch Agent

Used on EC2 to collect:

- Memory metrics.
- Process information.
- Disk metrics.

### AWS CloudTrail

Used for:

- AWS API audit trails.
- Tracking infrastructure changes.
- Investigating unexpected modifications.

---

## Security

### AWS IAM

Used for:

- Least-privilege access.
- Service roles.
- Resource permissions.

### AWS KMS

Used for:

- Encryption key management.
- Encryption of sensitive platform data where appropriate.

### AWS Secrets Manager

Used for:

- Application secrets.
- Webhook credentials.
- Other sensitive configuration.

### AWS WAF

May be used to protect the public application/API from common web attacks.

---

## Notifications

### Amazon SNS

Used for:

- Incident notifications.
- Recovery notifications.
- Escalation alerts.

SES may be considered for email delivery if direct email functionality is required.

---

# 28. AWS Deployment Security

The deployed platform must follow least-privilege principles.

Requirements:

- No unnecessary administrator permissions.
- Separate IAM roles for different components.
- Secrets must not be stored in source code.
- Sensitive data must use appropriate encryption.
- Public endpoints must be protected.
- CloudTrail should be enabled for auditability.
- S3 buckets should use appropriate public-access controls.
- DynamoDB should not be publicly accessible.

---

# 29. AWS Infrastructure as Code

The final AWS environment should be reproducible using Infrastructure as Code.

The project may use:

- AWS CDK
- AWS CloudFormation
- Terraform

The chosen IaC system should define as much of the final infrastructure as practical.

Example:

```text
Infrastructure as Code
│
├── Networking
├── IAM
├── DynamoDB
├── S3
├── CloudFront
├── API Gateway
├── Cognito
├── Lambda
├── EventBridge
├── CloudWatch
├── SNS
└── Other required resources
```

---

# 30. AWS Deployment Stages

The final deployment should be performed incrementally.

## Stage 1 — Core AWS Data

```text
DynamoDB
S3
IAM
```

## Stage 2 — Backend

```text
Lambda
API Gateway
```

## Stage 3 — Authentication

```text
Cognito
```

## Stage 4 — Monitoring

```text
CloudWatch
CloudWatch Agent
```

## Stage 5 — Eventing

```text
EventBridge
SQS where required
```

## Stage 6 — Healing Targets

```text
EC2
Lambda
S3
```

## Stage 7 — Healing Orchestration

```text
Step Functions where required
```

## Stage 8 — Frontend

```text
S3
CloudFront
```

## Stage 9 — Security / Audit

```text
IAM
KMS
Secrets Manager
CloudTrail
WAF where required
```

## Stage 10 — Notifications

```text
SNS
```

---

# 31. End-to-End AWS Flow

A complete EC2 recovery example:

```text
EC2 Application Failure
          ↓
CloudWatch
          ↓
CloudWatch Alarm
          ↓
EventBridge
          ↓
Healing Controller
          ↓
DynamoDB
Create Incident
          ↓
Healing Engine
          ↓
EC2 Adapter
          ↓
SSM
          ↓
Restart Application
          ↓
Verification
          ↓
DynamoDB
Update Incident
          ↓
SNS
Notification
```

Lambda example:

```text
Lambda Error Spike
        ↓
CloudWatch
        ↓
EventBridge
        ↓
Healing Engine
        ↓
DynamoDB Incident
        ↓
Lambda Adapter
        ↓
Rollback / Recovery
        ↓
Verification
        ↓
DynamoDB
        ↓
SNS
```

S3 example:

```text
S3 Policy Violation
        ↓
Detection
        ↓
EventBridge / Scheduled Evaluation
        ↓
Healing Engine
        ↓
DynamoDB Incident
        ↓
S3 Adapter
        ↓
Restore Configuration
        ↓
Verification
        ↓
DynamoDB
        ↓
SNS
```

---

# 32. Failure Simulation Requirements

The final AWS deployment must demonstrate real controlled failures.

## EC2

- Kill application process.
- Generate controlled memory pressure.
- Generate controlled disk pressure.

## Lambda

- Deploy a controlled failing version.
- Trigger abnormal error rate.
- Verify automatic recovery to known-good version.

## S3

- Introduce a controlled configuration violation.
- Verify automatic restoration.

All tests must be reversible and must not intentionally damage production data.

---

# 33. Observability

The final system should provide visibility into:

### Infrastructure

- EC2 health.
- Lambda health.
- S3 compliance.

### Platform

- Active incidents.
- Recovery attempts.
- Failed recoveries.
- Escalations.
- MTTR.
- Recovery success rate.

### Audit

- AWS API activity.
- Automated remediation actions.
- Policy changes.
- Incident state changes.

---

# 34. Recovery Metrics

The platform should calculate:

```text
Detection Time
Healing Start Time
Recovery Completion Time
```

Primary metric:

```text
MTTR =
Recovery Completion Time
-
Failure Detection Time
```

Additional metrics:

```text
Recovery Success Rate
Average Recovery Attempts
Incidents by Service
Incidents by Failure Type
Escalation Rate
```

---

# 35. Repository Structure

```text
aws-self-healing-infrastructure/
│
├── README.md
│
├── engine/
│   ├── event_processor.py
│   ├── healing_engine.py
│   ├── policy_engine.py
│   ├── verification.py
│   ├── incident_manager.py
│   └── state_manager.py
│
├── adapters/
│   ├── ec2/
│   │   ├── adapter.py
│   │   ├── diagnostics.py
│   │   └── remediation.py
│   │
│   ├── lambda/
│   │   ├── adapter.py
│   │   ├── diagnostics.py
│   │   └── remediation.py
│   │
│   └── s3/
│       ├── adapter.py
│       ├── diagnostics.py
│       └── remediation.py
│
├── models/
│   ├── resource.py
│   ├── incident.py
│   ├── healing_policy.py
│   └── recovery_attempt.py
│
├── api/
│   ├── resources.py
│   ├── incidents.py
│   ├── policies.py
│   └── metrics.py
│
├── frontend/
│
├── database/
│   ├── schema.md
│   └── access-patterns.md
│
├── policies/
│   ├── ec2/
│   ├── lambda/
│   └── s3/
│
├── scripts/
│   └── ec2/
│       ├── diagnose.sh
│       ├── restart-service.sh
│       ├── cleanup.sh
│       └── health-check.sh
│
├── infrastructure/
│   ├── iam/
│   ├── dynamodb/
│   ├── s3/
│   ├── cloudfront/
│   ├── api-gateway/
│   ├── cognito/
│   ├── lambda/
│   ├── eventbridge/
│   ├── cloudwatch/
│   ├── sns/
│   └── other/
│
├── tests/
│   ├── engine/
│   ├── adapters/
│   ├── api/
│   └── integration/
│
└── docs/
    ├── architecture.md
    ├── healing-policies.md
    ├── dynamodb-design.md
    ├── recovery-flows.md
    ├── aws-deployment.md
    └── incident-examples.md
```

---

# 36. Development Order

The project should be implemented in this order:

```text
1. Core Domain Models
        ↓
2. Healing Engine
        ↓
3. Policy Engine
        ↓
4. Incident State Machine
        ↓
5. Service Adapters
        ↓
6. Engine Tests
        ↓
7. DynamoDB Control Plane
        ↓
8. Backend API
        ↓
9. Operations Dashboard
        ↓
10. Authentication / Authorization
        ↓
11. Integration Testing
        ↓
12. AWS Infrastructure as Code
        ↓
13. AWS Deployment
        ↓
14. Real AWS Failure Simulation
        ↓
15. Observability / Hardening
```

The project should **not start with AWS deployment**.

The AWS environment is the final target environment for the completed platform.

---

# 37. MVP Definition

The MVP consists of three stages.

## Stage A — Engine MVP

```text
✓ Generic healing workflow
✓ Policy engine
✓ Incident state machine
✓ EC2 adapter
✓ Lambda adapter
✓ S3 adapter
✓ Verification
✓ Retry / escalation
✓ Idempotency
✓ Unit tests
```

## Stage B — Application MVP

```text
✓ DynamoDB control plane
✓ Resource management
✓ Policy management
✓ Incident management
✓ Recovery history
✓ Backend API
✓ Operations dashboard
✓ Authentication
✓ Integration tests
```

## Stage C — AWS MVP

```text
✓ AWS infrastructure
✓ EventBridge
✓ CloudWatch
✓ EC2
✓ Lambda
✓ S3
✓ SSM
✓ DynamoDB
✓ API Gateway
✓ Cognito
✓ CloudFront
✓ SNS
✓ IAM
✓ CloudTrail
✓ KMS / Secrets Manager where required
✓ Infrastructure as Code
✓ Real failure simulations
```

---

# 38. Definition of Done

The project is complete when:

- The generic healing engine works independently of deployment.
- EC2, Lambda, and S3 adapters are implemented.
- Each service has at least one detection policy.
- Each service has at least one remediation policy.
- Every remediation has a verification step.
- Incident state is persisted in DynamoDB.
- Concurrent healing attempts are prevented.
- Recovery attempts are recorded.
- The backend API is functional.
- The operations dashboard is functional.
- Authentication and authorization are implemented.
- The complete application is deployed on AWS.
- AWS infrastructure is reproducible through IaC.
- CloudWatch monitoring and EventBridge event processing work.
- Real controlled failures trigger the healing workflow.
- Successful recovery is verified automatically.
- Failed recovery is escalated.
- Notifications are generated.
- CloudTrail provides an audit trail.
- The project documents architecture, deployment, failure simulations, and recovery results.

---

# 39. Final Architecture

The completed platform should conceptually look like:

```text
                         USERS
                           │
                           ▼
                      CloudFront
                           │
                           ▼
                     S3 FRONTEND
                           │
                           ▼
                      API Gateway
                           │
                           ▼
                       Cognito
                           │
                           ▼
                    Backend / Lambda
                           │
                           ▼
                  DynamoDB Control Plane
                           │
                           │
                    ┌──────┴──────┐
                    │             │
                    ▼             ▼
              Application     Healing Engine
                                   │
                         ┌─────────┼─────────┐
                         ▼         ▼         ▼
                        EC2      Lambda      S3
                         │         │         │
                         ▼         ▼         ▼
                        SSM       APIs       APIs
                         │
                         └─────────┬─────────┘
                                   │
                                   ▼
                              Verification
                                   │
                                   ▼
                              DynamoDB
                                   │
                         ┌─────────┴─────────┐
                         ▼                   ▼
                        SNS              Dashboard
                         │
                         ▼
                    Notifications


Monitoring / Audit Layer
──────────────────────────────────────────
CloudWatch → EventBridge → Healing Engine
CloudTrail  → Audit Trail
IAM         → Authorization
KMS         → Encryption
Secrets Manager → Secrets
WAF         → Edge/API Protection
```

The core architectural principle remains:

> **Build the healing engine first, build the application around it second, and deploy the completed platform onto AWS last.**

AWS is therefore not the starting point of the project. **AWS is the final production environment in which the completed self-healing platform operates.**