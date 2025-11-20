
## Current Architecture Analysis
Trigger currently:
- Monitors database changes (insert, update, delete, replace)
- Processes documents and stores them in S3 (with chunking for large documents)
- Sends notifications to SQS for downstream ETL processing
- Runs serverlessly within Atlas environment

## Transition Strategy to Change Streams

### 1. Architecture Shift
**From:** Atlas Triggers (Serverless Functions)  
**To:** Application-based Change Streams Consumer

```
Current: MongoDB → Atlas Trigger → S3/SQS
New:     MongoDB → Change Stream → Application Service → S3/SQS
```

### 2. Implementation Plan

#### Phase 1: Setup Change Stream Infrastructure
```javascript
// New change stream consumer service
const { MongoClient } = require('mongodb');
const { S3Client } = require('@aws-sdk/client-s3');
const { SQSClient } = require('@aws-sdk/client-sqs');

class ChangeStreamProcessor {
  constructor(mongoUri, awsConfig) {
    this.client = new MongoClient(mongoUri);
    this.s3 = new S3Client(awsConfig);
    this.sqs = new SQSClient(awsConfig);
  }

  async watch(database, collection) {
    const pipeline = [
      {
        $match: {
          'operationType': { $in: ['insert', 'update', 'delete', 'replace'] }
        }
      }
    ];
    
    const changeStream = this.client
      .db(database)
      .collection(collection)
      .watch(pipeline, {
        fullDocument: 'updateLookup',
        resumeAfter: await this.getResumeToken() // Implement resume logic
      });

    changeStream.on('change', (change) => this.processChange(change));
  }

  async processChange(changeEvent) {
    // Port your existing S3/SQS logic here
    // Same chunking logic, same S3 structure
  }
}
```

#### Phase 2: Key Migration Components

| Component | Trigger Approach | Change Streams Approach |
|-----------|-----------------|------------------------|
| **Resume Logic** | Built-in | Implement with resume tokens |
| **Error Handling** | Atlas retry | Application-level retry |

### 3. Migration Steps

#### Step 1: Parallel Running
- Set up another repository for the stream processing which at first will run in parallel with triggers
- Deploy change stream consumer alongside existing trigger
- Monitor both for data consistency
- Compare S3 outputs and SQS messages

#### Step 2: Feature Parity Checklist
- All collection monitoring
- Error handling and retries
- Resume token persistence
- Monitoring and alerting

#### Step 3: Gradual Transition
1. We want to start with low-traffic collections
2. Validate data integrity
3. Migrate high-traffic collections
4. Disable triggers one by one


### 4. Resume Token Management
```javascript
// Implement persistent resume token storage
class ResumeTokenManager {
  async saveToken(token) {
    // Store in MongoDB or Redis
  }
  
  async getLastToken() {
    // Retrieve for resuming after restart
  }
}
```

### 5. Monitoring Strategy
- Track change stream lag
- Monitor document processing rate


# Setup steps
1. Create a workspace on mongo atlas
<img width="1383" height="336" alt="image" src="https://github.com/user-attachments/assets/6055e243-dee5-41e4-aa98-47e47a908004" />

2. Set up a repository where we will put the code for the stream functionality

### In the repository

Right now we use two functions in the triggers
<img width="268" height="94" alt="image" src="https://github.com/user-attachments/assets/d60ed822-84c0-4211-8ad3-29472fdd6dcc" />
One is responsible for getting the aws configuration

Second:
Captures the change event - Gets details about what changed (operation type, document data, collection name)
Uploads document data to S3 - Stores the full document (or document ID for deletes) as a JSON file in an S3 bucket

Handles large documents by splitting them into 4MB chunks if needed (Stitch has a 4MB limit)
Creates unique keys using: environment/collection/documentId-timestamp

Sends a message to SQS - Notifies a downstream ETL process by sending a message to an SQS queue that includes:

The operation type (insert/update/delete/replace)
References to the S3 file location(s) where the document data is stored
Uses the collection name as the message group ID
Uses the S3 key as a deduplication ID to prevent duplicate processing


### We could use the same approach with slight modifications
```javascript
const AWS = require('aws-sdk');
const { S3Client, PutObjectCommand } = require('@aws-sdk/client-s3');
const { SQSClient, SendMessageCommand } = require('@aws-sdk/client-sqs');

// AWS Configuration
const AWS_CONFIG = {
    region: process.env.AWS_REGION || 'us-east-1',
    credentials: {
        accessKeyId: process.env.AWS_ACCESS_KEY_ID,
        secretAccessKey: process.env.AWS_SECRET_ACCESS_KEY
    }
};

const s3 = new S3Client(AWS_CONFIG);
const sqs = new SQSClient(AWS_CONFIG);

// Environment variables
const S3_BUCKET_ETL_ENV = process.env.S3_BUCKET_ETL_ENV;
const S3Bucket = process.env.S3_ETL_BUCKET;
const SQS_QUEUE_ETL_URLS = JSON.parse(process.env.SQS_QUEUE_ETL_URLS || '{}');
const SHOULD_LOG_DEBUG_STATEMENTS = process.env.SHOULD_LOG_DEBUG_STATEMENTS === 'yes';

async function processChangeEvent(changeEvent) {
    console.log(`Operation type: ${changeEvent.operationType}`);
    
    if (changeEvent.operationType === 'update') {
        console.log(`Update description: ${JSON.stringify(changeEvent.updateDescription)}`);
    }
    
    if (!['insert', 'delete', 'replace', 'update'].includes(changeEvent.operationType)) {
        return;
    }

    const collection = changeEvent.ns.coll;
    const SQSQueueUrl = SQS_QUEUE_ETL_URLS[collection.toLowerCase()];

    if (SHOULD_LOG_DEBUG_STATEMENTS) {
        console.log('Using SQS Queue Url: ' + SQSQueueUrl);
    }

    const maxS3ObjectSizeInBytes = 4194304;
    let objectsToPutInS3 = [], objectsToPutInS3ForSQS = [];

    const fullStringBody = changeEvent.fullDocument 
        ? JSON.stringify(changeEvent.fullDocument) 
        : JSON.stringify(changeEvent.documentKey);

    if (SHOULD_LOG_DEBUG_STATEMENTS) {
        console.log('Before fullStringBodySize calc :' + Date.now());
        console.log('changeEvent.documentKey._id:' + changeEvent.documentKey._id);
    }

    const fullStringBodySize = fullStringBody ? fullStringBody.length : 0;
    const baseKeyName = `${S3_BUCKET_ETL_ENV}/${collection}/${changeEvent.documentKey._id}-${Date.now()}`;

    if (SHOULD_LOG_DEBUG_STATEMENTS) {
        console.log('fullStringBodySize=' + fullStringBodySize);
        console.log('baseKeyName=' + baseKeyName);
        console.log('maxS3ObjectSizeInBytes=' + maxS3ObjectSizeInBytes);
    }

    // Handle chunking for large documents
    if (fullStringBodySize > maxS3ObjectSizeInBytes) {
        const numChunks = Math.ceil(fullStringBodySize / maxS3ObjectSizeInBytes);
        const stringChunkLength = Math.ceil(fullStringBody.length / numChunks);
        objectsToPutInS3 = new Array(numChunks);
        objectsToPutInS3ForSQS = new Array(numChunks);
        
        for (let i = 0, n = 0; i < numChunks; i++, n += stringChunkLength) {
            const anObj = {
                Bucket: S3Bucket,
                Key: `${baseKeyName}-${i}`,
                Body: fullStringBody.substr(n, stringChunkLength)
            };
            objectsToPutInS3ForSQS[i] = anObj;
            objectsToPutInS3[i] = new PutObjectCommand(anObj);
        }
    } else if (fullStringBody) {
        const anObj = {
            Bucket: S3Bucket,
            Key: baseKeyName,
            Body: fullStringBody
        };
        objectsToPutInS3ForSQS.push(anObj);
        objectsToPutInS3.push(new PutObjectCommand(anObj));
    }

    if (SHOULD_LOG_DEBUG_STATEMENTS) {
        console.log('Pre-processing finished. Starting to write to S3');
    }

    // Write to S3
    const s3Promises = objectsToPutInS3.map(object => 
        s3.send(object).then(data => {
            console.log('S3 put object result: ' + JSON.stringify(data));
            return data;
        })
    );
    await Promise.all(s3Promises);

    console.log('Done with writing to S3. Time: ' + Date.now());

    // Prepare SQS message
    const sqsMsgBody = JSON.stringify({
        operation: changeEvent.operationType,
        S3FilePartsOfJSONDocument: objectsToPutInS3ForSQS.map(object => ({
            Bucket: object.Bucket, 
            Key: object.Key
        }))
    });

    if (SHOULD_LOG_DEBUG_STATEMENTS) {
        console.log('sqsMsgBody= ' + sqsMsgBody);
    }

    console.log(`SQSQueueUrl: ${SQSQueueUrl}`);
    console.log(`collection: ${collection}`);
    console.log(`baseKeyName: ${baseKeyName}`);

    // Send to SQS
    try {
        await sqs.send(new SendMessageCommand({
            QueueUrl: SQSQueueUrl,
            MessageGroupId: collection,
            MessageBody: sqsMsgBody,
            MessageDeduplicationId: baseKeyName
        }));
        console.log('Done with writing to SQS. Time: ' + Date.now());
    } catch (e) {
        console.log('Error writing to SQS: ' + e);
        throw e;
    }
}

// Change Stream setup
async function startChangeStream() {
    const { MongoClient } = require('mongodb');
    
    const client = new MongoClient(process.env.MONGODB_URI);
    
    try {
        await client.connect();
        console.log('Connected to MongoDB');
        
        const database = client.db(process.env.DB_NAME);
        const collection = database.collection(process.env.COLLECTION_NAME);
        
        // Or watch the entire database
        // const changeStream = database.watch();
        
        const changeStream = collection.watch();
        
        console.log('Watching for changes...');
        
        changeStream.on('change', async (changeEvent) => {
            try {
                await processChangeEvent(changeEvent);
            } catch (error) {
                console.error('Error processing change event:', error);
            }
        });
        
        changeStream.on('error', (error) => {
            console.error('Change stream error:', error);
        });
        
    } catch (error) {
        console.error('Failed to connect to MongoDB:', error);
        process.exit(1);
    }
}

// Start the change stream
startChangeStream();
```

###Key differences:
Configuration: Replace context.functions.execute() and context.environment.values with environment variables
Change Stream Setup: Added MongoDB client connection and change stream listener
Error Handling: Added proper error handling for the stream
Function Structure: Wrapped the logic in a processChangeEvent() function that gets called for each change event



## Summary
Transitioning from Atlas Triggers to Change Streams provides more control and flexibility but requires managing additional infrastructure. The migration should be gradual, with parallel running to ensure data consistency before full cutover.

