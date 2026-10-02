\# AWS Webhook Delivery Platform — Architecture



```mermaid

flowchart LR

&#x20;   	P\[Producer] --> API\[API Gateway]

&#x20;   	API --> ING\[Ingestion Lambda]

&#x20;   	ING --> DDB\[(DynamoDB)]



&#x20;   	DDB --> STREAM\[DynamoDB Streams]

&#x20;   	STREAM --> SP\[Stream Processor Lambda]

&#x09;SP --> MATCH\[Endpoint Matching]

&#x09;MATCH --> FAN\[Delivery Fan-Out]

&#x09;FAN --> SQS\[SQS Delivery Queue]

&#x20;   	SQS --> DW\[Delivery Worker]

&#x20;   	DW --> EP\[Customer Webhook Endpoint]



&#x20;   	DW --> DDB



&#x20;   	DW --> LOG\[Delivery Logs]

&#x20;   	LOG --> FH\[Kinesis Data Firehose]

&#x20;  	FH --> S3\[(S3)]

&#x20;   	S3 --> ATH\[Athena]



&#x20;   	DW --> CW\[CloudWatch]

&#x20;   	API --> XR\[X-Ray]

&#x20;   	DW --> XR



&#x20;   	SQS --> DLQ\[Dead-Letter Queue]

