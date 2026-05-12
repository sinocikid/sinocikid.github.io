---
title: "Serverless Computing: Functions vs Containers — When to Use Which"
date: 2026-05-16
categories: ["Cloud"]
tags: ["Serverless", "Cloud", "AWS Lambda", "Kubernetes", "Architecture"]
read_time: 8
---

"Serverless" is one of the most overloaded terms in cloud computing. It means you don't manage servers — until it doesn't, because Cloud Run and Fargate have servers underneath, they just abstract them away. The confusion is compounded by the fact that "serverless" now describes two meaningfully different execution models: Functions-as-a-Service (FaaS) and Container-as-a-Service (CaaS). Choosing the wrong one for your workload leads to unnecessary cost, complexity, or operational pain. This post cuts through the marketing to give you a practical framework for deciding which to use.

## What Serverless Actually Means

Serverless means the cloud provider manages server provisioning, patching, and scaling on your behalf. You provide code or containers; the platform handles the rest. The defining characteristics:

- **No capacity planning**: You don't provision fixed compute — the platform scales to demand, including to zero
- **Pay-per-use**: You pay for actual compute consumed, not idle capacity
- **Managed infrastructure**: OS patching, runtime updates, and hardware failure are the provider's problem

What serverless does NOT mean: no servers exist, no ops work is required, or it's always cheaper than alternatives. Serverless shifts operational burden from infrastructure management to platform constraints — and those constraints matter.

## FaaS: AWS Lambda, Azure Functions, Google Cloud Functions

Functions-as-a-Service is the most extreme form of serverless. You write a function — a single unit of code triggered by an event — and the platform executes it, scales it, and bills you per invocation.

**AWS Lambda** is the dominant player. Key characteristics:
- Invocations billed in 1ms increments
- Maximum execution time: 15 minutes
- Ephemeral local storage: 512MB–10GB in `/tmp`
- Up to 10GB memory, up to 6 vCPUs (proportional to memory)
- Concurrency: default 1,000 simultaneous executions per region (can be raised)

A simple Lambda handler in Python:

```python
import json
import boto3

def handler(event, context):
    # event contains the trigger payload
    order_id = event['detail']['order_id']

    # Process the event
    result = process_order(order_id)

    return {
        'statusCode': 200,
        'body': json.dumps({'processed': order_id})
    }
```

**Azure Functions** and **Google Cloud Functions** offer similar models with language-specific runtimes and cloud-native trigger integrations.

FaaS excels when:
- Work is **event-driven and discrete**: process a file upload, respond to a webhook, handle a queue message
- Execution is **short-lived**: seconds to a few minutes
- Traffic is **highly variable or spiky**: from zero to thousands of requests and back
- You want maximum operational simplicity and are willing to accept the constraints

## Container-as-a-Service: Fargate, Cloud Run, Azure Container Apps

CaaS is serverless for containers. You bring a container image; the platform runs it without you managing nodes.

**AWS Fargate**: Runs ECS or EKS pods without EC2 nodes to manage. You define a task definition with CPU and memory, and Fargate provisions the compute. Billed per vCPU-second and GB-second while the task runs. Fargate supports long-running services (always-on) and batch tasks.

**Google Cloud Run**: The cleanest CaaS experience. Deploy a container, Cloud Run scales from zero to many instances based on incoming HTTP traffic, and bills per request (100ms granularity). Supports session affinity, streaming, WebSockets, and gRPC. Minimum instances can be set to avoid cold starts.

**Azure Container Apps**: Runs containers with KEDA-based event-driven scaling, supporting HTTP scaling, queue-based scaling, and custom metrics. Includes a built-in Dapr sidecar option for service invocation and pub/sub.

CaaS excels when:
- You need **more than Lambda allows**: longer execution, larger runtime, custom native dependencies
- You're running a **web service or API** that should respond to HTTP traffic
- You want **containers without Kubernetes management overhead**
- Your workload has **moderate, predictable traffic** (not extreme spikiness)

## The Cold Start Problem

Cold starts are the most cited FaaS limitation. When a function hasn't been invoked recently (or a new concurrent instance is needed), the platform must:

1. Allocate a new execution environment
2. Download and unpack the function code or container image
3. Initialize the runtime (JVM warmup, Python interpreter, etc.)
4. Run any initialization code in the function handler

Cold start duration varies widely:

| Runtime | Typical Cold Start |
|---|---|
| Python | 100–400ms |
| Node.js | 100–400ms |
| Go | 100–200ms |
| Java (JVM) | 1–3 seconds |
| Java (GraalVM native) | 100–300ms |
| .NET 8+ NativeAOT | 100–200ms |

**Mitigation strategies:**

- **Provisioned Concurrency (Lambda)**: Pre-warm N execution environments that are always ready. You pay for them continuously, but cold starts for those instances are eliminated.
- **Minimum instances (Cloud Run)**: Keep at least one instance warm. Small cost, eliminates cold starts for base traffic.
- **Optimize package size**: Smaller deployment packages initialize faster. Use Lambda layers for shared dependencies.
- **Move runtime-heavy initialization outside the handler**: Database connections, SDK clients, and config loading done at module level (not inside the handler) are cached across invocations in the same execution environment.
- **Choose lightweight runtimes**: Go and Rust have near-zero startup overhead. Java with GraalVM native compilation eliminates JVM warmup.

For user-facing APIs where cold start adds noticeable latency, provisioned concurrency or minimum instances is the pragmatic fix.

## Cost Model: Event-Driven vs Always-On

FaaS cost is almost purely variable — you pay per invocation and per GB-second of execution. At low to medium traffic, this is almost always cheaper than always-on alternatives. At very high sustained traffic, the math shifts.

A rough comparison for a simple API handling 10M requests/month, each executing for 100ms with 256MB:

**AWS Lambda:**
- Requests: 10M × $0.0000002 = $2.00
- Compute: 10M × 0.1s × 256MB/1024 × $0.0000166667 = ~$4.17
- **Total: ~$6/month**

**Fargate (always-on, 2 tasks):**
- 2 × 0.25 vCPU × 730h × $0.04048/vCPU-hour = ~$14.80
- 2 × 0.5GB × 730h × $0.004445/GB-hour = ~$3.25
- **Total: ~$18/month**

**Lambda wins at this scale.** But at 100x the traffic with sustained load, Lambda compute costs grow linearly while Fargate stays flat. The crossover point is workload-specific — model it with your actual numbers before committing.

Additional cost factors often overlooked:
- **Egress costs**: Lambda and containers both incur data transfer costs
- **Cold start latency costs**: For user-facing APIs, cold starts affect user experience and potentially conversion rates
- **Idle cost**: FaaS is zero-cost when idle; CaaS with minimum instances has a floor cost

## State Management Challenges

Both FaaS and CaaS are fundamentally stateless. This is a feature — it enables scaling — but it requires explicit thought about state.

**In-memory state doesn't survive across invocations** in FaaS. Each Lambda invocation may run on a fresh execution environment. You cannot rely on global variables persisting (though they sometimes do — within the same execution environment). This rules out in-process caching of user sessions, work queues, or any state that must be consistent.

**Solutions:**
- **Databases for persistent state**: RDS, DynamoDB, Firestore — the obvious choice for data that must survive
- **Redis/ElastiCache for ephemeral shared state**: Distributed caches for session data, rate limiting counters, distributed locks
- **S3/GCS for large objects**: Files, intermediate processing artifacts
- **Step Functions / Workflows for orchestration state**: If a workflow spans multiple Lambda invocations, use a state machine to track progress rather than trying to pass state through invocation parameters

The stateless constraint forces good architectural habits. It makes services horizontally scalable by default and eliminates an entire class of distributed state bugs.

## Event-Driven Architectures with Serverless

FaaS shines in event-driven architectures. The pattern: something happens (a user uploads a file, an order is placed, a sensor sends data), and that event triggers processing. No polling, no always-on workers waiting for work.

Common event-driven patterns with Lambda:

```
S3 Upload → Lambda (image resize) → S3 (thumbnails)
API Gateway → Lambda (order validation) → SQS → Lambda (order processing) → DynamoDB
EventBridge (scheduled) → Lambda (daily report generation) → SES (email)
DynamoDB Streams → Lambda (search index update) → OpenSearch
```

The key insight: serverless functions compose naturally into event-driven pipelines. Each function does one thing, triggered by one event type. Scaling is automatic — if 10,000 files arrive simultaneously, Lambda spins up 10,000 concurrent executions (within concurrency limits).

SQS as a buffer between Lambda invocations is a particularly useful pattern — it decouples producers from consumers, provides retry logic, and handles backpressure automatically.

## Decision Matrix

Use this to guide your architecture decisions:

| Criterion | FaaS (Lambda/Functions) | CaaS (Fargate/Cloud Run) | Kubernetes |
|---|---|---|---|
| Execution duration | < 15 min | Hours/continuous | Unlimited |
| Traffic pattern | Spiky / zero-to-N | Variable but sustained | Predictable / high |
| Custom dependencies | Limited | Full container | Full container |
| Cold start tolerance | Depends on use case | Low (min instances) | Not applicable |
| Operational overhead | Minimal | Low | High |
| Max unit of scale | Function | Container instance | Pod/node |
| Stateful workloads | No | Limited | Yes |
| Cost at high sustained load | Can be expensive | Moderate | Efficient |
| Multi-step workflows | Step Functions needed | In-process possible | Flexible |

## Real-World Use Cases

**FaaS is the right choice for:**
- Webhook processors (GitHub, Stripe, Twilio callbacks)
- Scheduled jobs (nightly reports, cleanup tasks)
- File processing pipelines (image resizing, document parsing on upload)
- Lightweight API backends for mobile apps with variable traffic
- Stream processing (Kinesis, DynamoDB Streams consumers)

**CaaS is the right choice for:**
- Web APIs and microservices that need container portability
- Long-running batch jobs exceeding Lambda's 15-minute limit
- Applications with complex native dependencies (ML inference, video processing)
- Services requiring WebSocket connections or server-sent events
- Teams already using Docker who want serverless benefits without FaaS constraints

**Kubernetes is the right choice for:**
- Multi-tenant platforms with complex scheduling requirements
- Workloads requiring persistent storage with specific performance characteristics
- Applications needing fine-grained control over networking and security
- Stateful services (databases, message brokers) run as containerized workloads

## Conclusion

The FaaS vs CaaS decision isn't binary, and real architectures often use both. Your event-driven background processing might be Lambda; your user-facing API might be Cloud Run; your data platform might be Kubernetes. Each execution model has a sweet spot.

The core question to ask: "What does this workload look like in terms of duration, trigger type, and traffic pattern?" Short, event-triggered, spiky — FaaS. Container-based, HTTP-driven, moderate scale — CaaS. Complex, stateful, high-throughput — Kubernetes. Don't let hype drive the decision. Model the cost, test the cold starts, and match the tool to the actual workload shape.
