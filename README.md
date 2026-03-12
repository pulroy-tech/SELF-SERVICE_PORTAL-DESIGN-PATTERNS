# SELF-SERVICE_PORTAL-DESIGN-PATTERNS


1. **The Singleton pattern**
   
What it is?
Singleton is a creational pattern that guarantees:

There is only one instance of a type in the program.
That instance is globally accessible (within the process).
So “singleton” = one instance, shared everywhere.

Why use it
Typical reasons:

Expensive to create (e.g. DB/S3/HTTP clients, connection pools): create once, reuse.
Shared state (e.g. config, cache): one source of truth.
External resource (e.g. one connection to a service): avoid many connections when one is enough.
Classic structure (conceptually)
┌─────────────────────────────────────────┐
│  Singleton                               │
├─────────────────────────────────────────┤
│  - instance (private, static)            │  ← single instance
│  - getInstance() (public)                │  ← only way to get it
└─────────────────────────────────────────┘

Rules:
Private constructor (or no public way to create multiple instances).
One shared instance (stored in a package-level / static variable).
Single access point (e.g. getInstance() or getXxxClient()).
In Go specifically
Go has no “private constructor,” so the pattern is done by convention:

Package-level variable holds the single instance (e.g. dynamoClient, s3Client).
One function (e.g. getDynamoDBClient(), getS3Client()) is the only place that creates and returns that instance.
Thread safety: if creation can happen from multiple goroutines, use a mutex so only one goroutine creates the instance (double-checked or “init under lock” style).
That’s exactly what your backend does.

