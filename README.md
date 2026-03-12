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



2.** How it’s implemented in your project**


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
var (
	s3Client *s3.Client
	s3Mu     sync.Mutex
)

s3client.go
Lines 22-55
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
Singleton: One s3.Client for the process.
Lazy: Created on first use (e.g. from uploadToS3, getObjectFromS3).
Thread-safe: s3Mu protects the “if nil then create” section.
2.3 How the rest of the code uses it
Nothing creates DynamoDB or S3 clients directly. They always go through the getter:

DynamoDB: putSpecSubmission, getSpecSubmission, getSpecSubmissionsByUser, updateSpecSubmission all call getDynamoDBClient() and then use that client.
S3: uploadToS3 and getObjectFromS3 call getS3Client() and then use that client.
So the “single access point” rule is satisfied: one function per client, one instance per process.


