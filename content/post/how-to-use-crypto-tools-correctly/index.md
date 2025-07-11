---
title: "Stop Coding, Start Thinking: The Mental Framework To Thinking Like An Software Architect" 
description: 
slug: test
# image: localstorage-token.png
categories:
  - system-design
tags:
  - System Design
  - Use Case
  - Software Architecture
weight: 1 # You can add weight to some posts to override the default sorting (date descending)
---

You're given a new project. The requirements are complex: "We need a system that allows users to upload sensitive documents. It must scale to 100 uploads per second, the process must be fast for the user, and the documents must pass OCR and validation before the user approves them. Oh, and it must be secure and highly available."

### What is your first reaction?

If your brain immediately jumps to "Okay, I'll use **Node.js with Express and multer for the upload, and maybe a PostgreSQL table for the status**," you're falling into the most common development trap: **thinking about the technology before thinking about the problem**.

The difference between someone who just codes and an architect (or senior developer) isn't the number of frameworks they know. It's the existence of a mental framework to break down chaos and transform it into a coherent design.

This article will teach you this framework. It's a four-step process for translating complex requirements into a robust architecture, before you even write the first line of code.

# Our Case Study: Creating a KYC (Know Your Customer)

Imagine we've been tasked with building the backend for an identity verification process. The requirements are as follows:

#### Functional Requirements:

Functional Requirements:

- The user, through our mobile app, must be able to upload a photo of an identification document (e.g., a driver's license).

- The system must process this image, extracting data (name, ID number, etc.) via OCR (Optical Character Recognition).

- The system needs to validate the extracted data with an external government service.

- The user's status must be updated to "Verified" or "Rejected."

#### Non-Functional Requirements (Where our focus will be):

- Scalability: The system must handle normal new user flows and marketing campaign peaks. We estimate peaks of up to 100 uploads per second.

- Latency: The user must receive an "upload received successfully" confirmation in under 2 seconds. The final verification result can take a few minutes.

- Security: We are dealing with sensitive personal documents (PII - Personally Identifiable Information). Security is critical. We need to ensure data confidentiality and integrity, both in transit and at rest, in compliance with regulations like GDPR/LGPD.

- Availability: The upload functionality must be highly available. The processing itself can tolerate minor failures and recover.

# Step 1: Ignore the Technology. Break the Problem Down into "Verbs"

The first step is to resist the urge to code. Read the requirements and extract the essential actions, the system's "verbs." Tell the story of what needs to happen, in a simple and linear fashion. You don't start building a house by choosing the brand of the hammer; you understand the steps: lay the foundation, raise the walls, install the roof.

For our KYC system, the story is as follows:

1. The user needs to **UPLOAD** a file.
2. The system needs to **STORE** this file securely.
3. The system needs to **NOTIFY** other parts that a new file has arrived.
4. A process needs to **READ** the newly arrived file.
5. This process needs to **PROCESS** the image (OCR).
6. This process needs to **VALIDATE** the extracted data with an external service.
7. The system needs to **UPDATE** the user's status based on the result.

See the magic. We've traded a complex paragraph for a clear and discrete chain of events. Each verb is a link in the chain, and now we can reason about each of them in isolation, which is infinitely easier.

# Step 2: Identify the "Forces" (Constraints) at Play

Now, we look at the non-functional requirements. They aren't a wish list; they are the forces of nature that will push, pull, and bend our architecture. A good architect doesn't fight these forces; they use them to shape the design.

For each force, your brain should set off an alarm:
 - **Force of Scalability + Latency: "100 uploads/second" and "confirmation in < 2s".**: Alarm bells should be going off in your head: "It's impossible to manage this in a single monolithic API. The I/O bottleneck from receiving 100 concurrent files will saturate anything. And the full processing (OCR, validation) will definitely take longer than 2 seconds. Therefore, the user's action (the upload) must be decoupled from everything else. The user flow cannot be synchronous."

 - **Force of Availability + Resilience: "Upload always available" and "processing can fail".**: Alarm bells should be going off in your head: "What if the OCR service is down for 5 minutes? Under no circumstances can this prevent new uploads. This means there must be a 'buffer', a shock absorber between the entry point and the processors. If a processor dies, another one must be able to pick up the work from the queue."

 - **Force of Security: "Sensitive documents, GDPR/LGPD".**: Alarm bells should be going off in your head: "Every point where this data touches is a legal and reputational liability. How does it travel over the network (encryption in transit)? How is it stored on disk (encryption at rest)? Who can read the original file? Who can see the OCR result? I need audit trails for every access."

# Step 3: Map "Forces" to "Design Patterns"

This is where seniority pays off. Experience is largely about having a mental catalog of solutions to recurring problems. You don't reinvent the wheel; you recognize which type of terrain calls for which type of wheel.

Let's map the intuitions from Step 2 to well-known architectural patterns:
- **The force of "high-throughput ingress + decoupled slow processing"** -> maps directly to the **Message Queue** pattern. You instantly think: "This is a classic use case for SQS, RabbitMQ, or Kafka. The upload publishes a message, and workers consume it at their own pace."

- **The force of "heavy file upload + not overwhelming the API"** -> maps to the **Direct Upload to Object Storage (via Presigned URLs)** pattern. You think: "I don't want these megabytes passing through my API. The pattern is to offload this directly to S3, Google Cloud Storage, or Azure Blob Storage. My API only needs to do the lightweight job of generating a temporary, secure upload link."

- **The force of "data security + granular access control"** -> maps to the **Server-Side Encryption (SSE) and Principle of Least Privilege (IAM Roles)** patterns. You think: "S3 solves the storage, but how do I lock it down? Provider-managed encryption is the standard. And the worker that performs OCR must only have READ permission, while the upload service only has WRITE permission."

- **The force of "multiple steps in an asynchronous workflow"** -> maps to the **State Machine or Saga** pattern. You think: "I need to control this flow from `'UPLOADED'` -> `'PROCESSING'` -> `'VALIDATED'` -> `'FAILED'`. A state machine like AWS Step Functions is the way to organize this asynchronous chaos and ensure no file gets lost."

What did we do here? We didn't invent anything. We merely recognized the problems for which robust, industry-tested solutions already exist.

# Step 4: Always Think in Trade-offs

This is the step that separates architects from mere implementers. No choice is free. For every pattern you adopt, you must ask yourself honestly: **"What's the cost?"**

- "**Okay, I chose a Message Queue.** What's the trade-off? I've added operational complexity. My system is now distributed. I need to worry about workers, about messages that fail and go to a Dead-Letter Queue (DLQ), and the flow becomes asynchronous, which means eventual consistency. Does the benefit (scalability and resilience) outweigh this cost? **Yes, for this problem, absolutely.**"

"**Okay, Presigned URLs.** What's the trade-off? The client-side logic (mobile/web app) gets slightly more complex. It first needs to call my API to get the URL and only then perform the upload to S3. It's one extra API call. Does the benefit (saving my API from bandwidth and processing saturation) outweigh this cost? **Yes, without a doubt.**"

# Conclusion: From Confusion to Clarity

Observe what happened. We went from a nebulous set of requirements to a clear and well-defined architecture:

- A mobile client requests a **Presigned URL** from a lightweight API endpoint.

- The client uploads the file directly to an S3 bucket, which is configured with **server-side encryption**.

- Upon receiving the new object, S3 triggers an event that creates a message in an **SQS Queue**.

- A group of workers (services or Lambda functions) with **restricted IAM permissions** consume messages from the queue.

- A **State Machine** orchestrates the flow of each file through the OCR and external validation workers, finally updating the user's status in the database.

This is the mental framework that turns developers into architects. It moves the discussion from technology to the problem, from tools to principles. Practice it, and you will no longer be just writing code; you will be designing resilient, scalable, and secure systems on purpose.