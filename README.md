# SELF-SERVICE_PORTAL-DESIGN-PATTERNS


### 1. The Singleton pattern**
   
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



### 2. How it’s implemented in project


You use the lazy singleton pattern (create on first use) for two shared clients, with a mutex for thread-safe one-time initialization.
2.1 DynamoDB client (backend/dynamodb.go)
Single instance + lock:


dynamodb.go
Lines 18-22
```
var (
	dynamoClient *dynamodb.Client
	dynamoMu     sync.Mutex
)
```


Single accessor (lazy init under lock):
dynamodb.go
Lines 24-56
```
func getDynamoDBClient() (*dynamodb.Client, error) {
	dynamoMu.Lock()
	defer dynamoMu.Unlock()
	if dynamoClient != nil {
		return dynamoClient, nil
	}
	region := os.Getenv("AWS_REGION")
	// ... config and optional custom endpoint ...
	dynamoClient = dynamodb.NewFromConfig(cfg, opts...)
	log.Info("dynamodb: client initialized")
	return dynamoClient, nil
}
```


Singleton: Only dynamoClient is ever set; all callers get the same pointer.
Lazy: Created the first time getDynamoDBClient() is called (e.g. from putSpecSubmission, getSpecSubmission, getSpecSubmissionsByUser, updateSpecSubmission).
Thread-safe: dynamoMu ensures only one goroutine runs the “create if nil” block, so you don’t get two clients or a data race.
2.2 S3 client (backend/s3client.go)
Same idea:


s3client.go
Lines 17-21
```
var (
	s3Client *s3.Client
	s3Mu     sync.Mutex
)
```
s3client.go
Lines 22-55
```
func getS3Client() (*s3.Client, error) {
	s3Mu.Lock()
	defer s3Mu.Unlock()
	if s3Client != nil {
		return s3Client, nil
	}
	// ... load config, optional AWS_ENDPOINT_URL ...
	s3Client = s3.NewFromConfig(cfg, opts...)
	log.Info("s3: client initialized")
	return s3Client, nil
}
```
Singleton: One s3.Client for the process.
Lazy: Created on first use (e.g. from uploadToS3, getObjectFromS3).
Thread-safe: s3Mu protects the “if nil then create” section.
2.3 How the rest of the code uses it
Nothing creates DynamoDB or S3 clients directly. They always go through the getter:

DynamoDB: putSpecSubmission, getSpecSubmission, getSpecSubmissionsByUser, updateSpecSubmission all call getDynamoDBClient() and then use that client.
S3: uploadToS3 and getObjectFromS3 call getS3Client() and then use that client.
So the “single access point” rule is satisfied: one function per client, one instance per process.



## Chain of Responsibility

### 1. What It Is?
Chain of Responsibility is a behavioral design pattern: a request is passed along a chain of handlers. Each handler can either:

Handle the request (and optionally stop),
Pass the request to the next handler,
Or do both (e.g. run logic, then pass on).
So responsibility is split across a chain instead of one big handler.

### 2. Intent and Problem It Solves
Intent: Decouple sender and receiver. Multiple objects can handle the request; the request moves along the chain until someone handles it (or it reaches the end).
Problem: You want to process a request in several steps (e.g. logging → auth → business logic) without hard‑coding a single “god” function that does everything. You want to add/remove/reorder steps without changing the core logic.

### 3. How It Works (Step by Step)
Request is given to the first handler.
Handler can:
Do work (e.g. set headers, validate token, log).
Call the next handler (pass the request along).
Stop (e.g. send 401 and don’t call next).
This repeats for each handler in order.
Often the last handler is the one that does the “real” work (e.g. your route handler); earlier ones are “middleware” (security, auth, logging).
So the chain is a pipeline: Request → Handler1 → Handler2 → … → Final handler.

### 4.Chain of Responsibility in our project

Idea: Every HTTP request goes through a fixed order of handlers. Each handler does its job, then either calls the next or stops (e.g. returns an error).

In our app:
Gin’s default middleware (logger, recovery) runs first.
Security headers (setupSecurityHeaders) run next: they set headers and always call c.Next() so the request continues.
JWT authorizer (where used, e.g. for /routes): validates the token. If invalid → responds with 401 and does not call c.Next() (chain stops). If valid → calls c.Next().
Your route handler (e.g. requestsProvider, specUploadHandler, apiDiscovery) runs last and does the actual work (DB, S3, Kong, etc.).
So: Request → Security → (optional) Auth → Your handler. Each step either passes the request to the next with c.Next() or ends the chain by not calling it. That’s Chain of Responsibility: a pipeline of handlers, each with one responsibility, in order.

<img width="1226" height="672" alt="image" src="https://github.com/user-attachments/assets/7a00424c-bb90-46f6-86b9-9f083e2f6245" />


## FACADE 

What is a Facade?
Facade is a structural pattern: one simple interface in front of many subsystems.

Problem: The client would need to know and call several complex pieces (different APIs, DBs, configs).
Idea: Introduce one entry point (the “facade”) that:
Exposes a small, easy-to-use API.
Hides how many subsystems exist and how they’re called.
Does the orchestration internally (calling the right services in the right order).
So: one simple front door → behind it, many parts working together.

In your project: the backend is the facade
The Go backend is that single front door for the React app. The frontend only talks to the backend; the backend talks to everything else and returns simple responses.

<img width="620" height="442" alt="image" src="https://github.com/user-attachments/assets/50d2f574-c692-4572-9c94-0937690d87ed" />


### How the facade behaves on a few flows

1. “My requests” — GET /requests
Frontend: One call: GET backend_url/requests with the JWT.
Facade (backend):
Validates JWT and gets user id.
Reads submissions from DynamoDB (by user).
Calls Jira to get current status for each ticket.
Builds a single JSON list (e.g. requests + total) and returns it.
The frontend doesn’t know about DynamoDB or Jira; it only sees “give me my requests” and one response.

2. “Submit spec” — POST /specs
Frontend: One call: POST backend_url/specs with FormData (topic, description, status, spec file).
Facade (backend):
Validates JWT and form.
Creates a ticket in Jira.
Uploads the file to S3 (e.g. user-id/ticket-id/openapi-spec.json).
Writes a record in DynamoDB.
Returns something like { ticket_key, message }.
Again: one endpoint, one place to send the token; Jira + S3 + DynamoDB are hidden behind the facade.

3. “List APIs / routes” — GET /routes
Frontend: GET backend_url/routes (with optional query params).
Facade (backend):
Validates JWT.
Calls Kong Admin (routes, services).
Can filter/merge and shape the data.
Returns a single JSON (e.g. list of routes with service info).
The frontend doesn’t talk to Kong directly; the backend is the only place that does.

4. “Check spec” — POST /apiInsights/analyze
Frontend: POST backend_url/apiInsights/analyze with the spec file.
Facade (backend):
Validates JWT.
Sends the file to API Insights (or whatever service does the analysis).
Returns the analysis result in one response.
So again: one simple call, one response; the real service is behind the facade.



