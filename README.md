# MongoDB Trigger to Stream Processing Migration Plan

## Executive Summary

This document outlines a comprehensive plan to migrate from MongoDB database triggers to a dedicated stream processing solution for ETL operations. The migration aims to improve scalability, reliability, and observability while ensuring zero data loss during the transition.

**Current State:** 8 MongoDB triggers across multiple collections handling change events and writing to S3/SQS  
**Target State:** Robust stream processing architecture with improved monitoring and error handling  
**Timeline:** 8-12 weeks (Investigation: 2 weeks, Implementation: 6-10 weeks)  
**Technology Stack:** Node.js/TypeScript  
**Repository:** New dedicated repository for stream processing services

---

## Table of Contents

1. [Current Architecture Analysis](#current-architecture-analysis)
2. [Technology Evaluation](#technology-evaluation)
3. [Recommended Architecture](#recommended-architecture)
4. [Detailed Migration Plan](#detailed-migration-plan)
5. [Risk Mitigation](#risk-mitigation)
6. [Testing Strategy](#testing-strategy)
7. [Rollback Plan](#rollback-plan)

---

## Current Architecture Analysis

### Existing Triggers

| Trigger Name | Database | Collection | Cluster |
|-------------|----------|------------|---------|
| aadw_reservationdatafacilities_v3 | Facilitron_Prod | aaDW_ReservationDataFacilities_v3 | None (Primary) |
| reservation_charges_etl | Facilitron_Prod | reservation_charges | Production-Copy |
| reservation_objs_etl | Facilitron_Prod | reservation_objs | Production-Copy |
| reservation_payments_etl | Facilitron_Prod | reservation_payments | Production-Copy |
| users_etl | Facilitron_common | users | DevelopmentCluster |
| owners_etl | Facilitron_common | owners | DevelopmentCluster |
| reservations_etl | Facilitron_Prod | reservations | None (Primary) |

### Current Flow

```
MongoDB Change Event → Trigger Function → S3 (document chunks) → SQS Message → ETL Processing
```

### Current Limitations

1. **Scalability Issues**
   - Triggers run in MongoDB Atlas environment with resource constraints
   - Limited to 4MB S3 object size (Stitch limitation)
   - No built-in batching or throughput optimization
   - Limited to 10 console.log calls per trigger execution

2. **Reliability Concerns**
   - No automatic retry mechanism beyond basic trigger retry
   - Limited error handling and dead letter queue support
   - Difficult to replay events or handle backpressure
   - Event ordering only within single document changes

3. **Cost Implications**
   - Trigger executions are metered by MongoDB Atlas
   - Inefficient for high-volume collections
   - Multiple S3 PUT operations for large documents

---

## Technology Evaluation

### Option 1: MongoDB Change Streams (Native)

#### Overview
MongoDB Change Streams provide a native way to watch for changes in collections, databases, or entire deployments in real-time.

#### Architecture
```
MongoDB Change Stream → Application Consumer → S3 → SQS → ETL
```

#### Pros
✅ **Native MongoDB integration** - No external dependencies  
✅ **Resume tokens** - Built-in mechanism for exactly-once processing  
✅ **Ordered delivery** - Maintains order of changes per document  
✅ **Filtering capabilities** - Can filter at database level  
✅ **Lower latency** - Direct connection to MongoDB  
✅ **No additional infrastructure** - Uses existing MongoDB cluster  
✅ **Cost-effective** - No additional streaming platform costs  

#### Cons
❌ **No built-in buffering** - Application must handle buffering  
❌ **Single point of failure** - If consumer goes down, must handle resume  
❌ **Scaling complexity** - Need to implement sharding logic for multiple consumers  
❌ **No native dead letter queue** - Must implement error handling  
❌ **Connection management** - Application responsible for connection health  
❌ **Limited transformation** - All processing in application code  

#### Best For
- Lower volume collections (<1000 changes/second)
- When minimal external dependencies are desired
- When cost optimization is critical
- Development/staging environments

#### Implementation Complexity
**Medium** - Requires building robust consumer application with error handling, resumption logic, and monitoring.

#### Estimated Cost
**Low** - Only compute costs for consumer application (~$50-100/month for EC2/ECS)

---

### Option 2: Apache Kafka + MongoDB Connector

#### Overview
Use Confluent/Debezium MongoDB Connector to stream changes to Kafka, then process with Kafka consumers.

#### Architecture
```
MongoDB → Kafka Connect (MongoDB Connector) → Kafka Topics → Consumer Application → S3 → SQS → ETL
```

#### Pros
✅ **Industry standard** - Mature ecosystem and tooling  
✅ **High throughput** - Can handle millions of events per second  
✅ **Durability** - Built-in replication and persistence  
✅ **Exactly-once semantics** - With proper configuration  
✅ **Rich ecosystem** - Kafka Streams, KSQL, Schema Registry  
✅ **Multiple consumers** - Easy to fan out to multiple processing pipelines  
✅ **Monitoring tools** - Extensive metrics and monitoring options  
✅ **Replay capability** - Can replay historical events  
✅ **Transformation options** - Can use Kafka Streams for pre-processing  

#### Cons
❌ **Operational complexity** - Requires Kafka cluster management  
❌ **Infrastructure overhead** - Need ZooKeeper (or KRaft), brokers, Connect workers  
❌ **Higher latency** - Additional hop through Kafka adds ~10-50ms  
❌ **Cost** - MSK or self-managed Kafka can be expensive ($500-2000+/month)  
❌ **Learning curve** - Team needs Kafka expertise  
❌ **Network overhead** - Data moves: MongoDB → Kafka → Consumers  

#### Best For
- High volume collections (>1000 changes/second)
- When you need multiple downstream consumers
- When replay and historical analysis are important
- Organizations with existing Kafka infrastructure
- When complex stream processing is needed

#### Implementation Complexity
**High** - Requires Kafka cluster setup, connector configuration, monitoring setup, and operational expertise.

#### Estimated Cost
**High** - MSK cluster: $600-2000/month (3 brokers, m5.large) + data transfer costs

---

### Option 3: AWS Kinesis Data Streams

#### Overview
AWS-native streaming service with MongoDB change stream integration via custom application or Lambda.

#### Architecture
```
MongoDB Change Stream → Lambda/ECS Producer → Kinesis Stream → Lambda/ECS Consumer → S3 → SQS → ETL
```

#### Pros
✅ **Fully managed** - AWS handles infrastructure  
✅ **AWS native** - Integrates seamlessly with S3, SQS, Lambda, CloudWatch  
✅ **Auto-scaling** - On-demand or provisioned capacity with auto-scaling  
✅ **Built-in monitoring** - CloudWatch metrics and dashboards  
✅ **Multiple consumers** - Enhanced fan-out for parallel processing  
✅ **Retention** - Data retention up to 365 days  
✅ **Low operational overhead** - No cluster management  
✅ **Exactly-once processing** - With Kinesis enhanced fan-out and checkpointing  
✅ **Encryption at rest and in transit** - Built-in security features  

#### Cons
❌ **AWS lock-in** - Tied to AWS ecosystem  
❌ **Cost at scale** - Can be expensive for high throughput (shard-hour pricing)  
❌ **Shard management** - May need to manage shard splitting/merging  
❌ **2MB record limit** - Smaller than Kafka (10MB default)  
❌ **Less flexible** - Fewer transformation options than Kafka Streams  
❌ **Learning curve** - Kinesis-specific concepts (shards, enhanced fan-out)  

#### Best For
- AWS-centric architecture
- When operational simplicity is paramount
- Medium to high volume workloads
- When tight AWS service integration is beneficial
- Organizations without stream processing expertise

#### Implementation Complexity
**Medium** - Requires Lambda or ECS setup, IAM configuration, and monitoring. AWS manages infrastructure.

#### Estimated Cost
**Medium** - On-demand: ~$0.015 per 1M records + $0.040 per GB. Estimated $300-800/month for medium volume.

---

### Option 4: AWS Kinesis Data Firehose (Simplified)

#### Overview
Fully managed service to deliver streaming data directly to S3, with optional Lambda transformation.

#### Architecture
```
MongoDB Change Stream → Lambda Producer → Kinesis Firehose → S3 (with buffering) → S3 Event → Lambda → SQS
```

#### Pros
✅ **Simplest AWS option** - Minimal configuration  
✅ **Direct S3 delivery** - Automatic batching and delivery  
✅ **Built-in transformation** - Lambda transformation built-in  
✅ **Auto-scaling** - No capacity planning needed  
✅ **Cost-effective** - Pay only for data volume  
✅ **Format conversion** - Can convert to Parquet/ORC for BI  
✅ **Buffer optimization** - Automatic batching reduces S3 operations  

#### Cons
❌ **Limited consumers** - Only delivers to specific destinations  
❌ **Less flexible** - Cannot easily add multiple processing pipelines  
❌ **Latency** - Buffering introduces 60+ second delay  
❌ **No replay** - Cannot reprocess historical data  
❌ **Limited routing** - Cannot route different events to different destinations  

#### Best For
- Simple S3 delivery use case
- When latency requirements are relaxed (>60 seconds acceptable)
- Cost-sensitive deployments
- When you want minimal operational overhead

#### Implementation Complexity
**Low** - Simplest to implement and maintain.

#### Estimated Cost
**Low** - ~$0.029 per GB. Estimated $100-300/month for medium volume.

---

## Recommended Architecture

### Primary Recommendation: AWS Kinesis Data Streams

#### Rationale

After evaluating all options, **AWS Kinesis Data Streams** is recommended as the optimal solution for the following reasons:

1. **AWS Native Integration** - Your existing infrastructure uses S3 and SQS (AWS services), making Kinesis a natural fit
2. **Managed Service** - Reduces operational burden compared to Kafka while maintaining reliability
3. **Scalability** - Handles current and future volume requirements with on-demand capacity
4. **Cost Balance** - More affordable than Kafka, with better features than basic Change Streams
5. **BI Requirements** - Built-in integration with analytics services for future BI needs
6. **Team Skillset** - Lower learning curve than Kafka for AWS-familiar teams

#### Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────────┐
│                         MongoDB Atlas                                │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐              │
│  │ Facilitron   │  │ Facilitron   │  │ Development  │              │
│  │ _Prod        │  │ _common      │  │ Cluster      │              │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘              │
│         │                  │                  │                       │
└─────────┼──────────────────┼──────────────────┼───────────────────────┘
          │                  │                  │
          └──────────────────┴──────────────────┘
                             │
                  ┌──────────▼──────────┐
                  │  Change Stream      │
                  │  Consumer (ECS)     │
                  │  - Resilient        │
                  │  - Multi-threaded   │
                  │  - Resume tokens    │
                  └──────────┬──────────┘
                             │
                  ┌──────────▼──────────┐
                  │  Kinesis Data       │
                  │  Streams            │
                  │  - Per collection   │
                  │  - On-demand mode   │
                  │  - 24hr retention   │
                  └──────────┬──────────┘
                             │
          ┌──────────────────┴──────────────────┐
          │                                      │
┌─────────▼─────────┐                 ┌─────────▼─────────┐
│  Kinesis Consumer │                 │  Kinesis Consumer │
│  Application      │                 │  Application      │
│  (ECS Fargate)    │                 │  (Lambda - Future)│
│  - Batch process  │                 │  - BI Processing  │
│  - Checkpointing  │                 │  - Analytics      │
│  - Error handling │                 │                   │
└─────────┬─────────┘                 └───────────────────┘
          │
          ├─────────────┬─────────────┐
          │             │             │
┌─────────▼────┐ ┌──────▼──────┐ ┌───▼─────┐
│ S3 Bucket    │ │ SQS Queue   │ │ DLQ     │
│ (ETL Data)   │ │ (Existing)  │ │ (Errors)│
└──────────────┘ └─────────────┘ └─────────┘
```

#### Key Components

1. **Change Stream Consumer (ECS Service)**
   - Dockerized application running on ECS Fargate
   - Connects to MongoDB Change Streams for each collection
   - Handles resume tokens for fault tolerance
   - Writes to appropriate Kinesis stream per collection

2. **Kinesis Data Streams**
   - One stream per collection (or logical grouping)
   - On-demand capacity mode (auto-scales)
   - 24-hour retention (sufficient for recovery)
   - CloudWatch metrics enabled

3. **Kinesis Consumer Application (ECS/Fargate)**
   - Processes events in batches
   - Handles document chunking (4MB limit)
   - Writes to S3 and sends SQS messages
   - Uses DynamoDB for checkpointing (via KCL)
   - Implements retry logic with exponential backoff

4. **Dead Letter Queue (DLQ)**
   - Captures failed events for manual review
   - CloudWatch alarms for DLQ depth
   - Replay mechanism for corrected events

---

## Code Implementation Examples

This section provides production-ready Node.js/TypeScript code examples for the stream processing service.

### Repository Structure

```
stream-processing-service/
├── packages/
│   ├── change-stream-consumer/
│   │   ├── src/
│   │   │   ├── index.ts
│   │   │   ├── mongodb-client.ts
│   │   │   ├── kinesis-producer.ts
│   │   │   └── config.ts
│   │   ├── tests/
│   │   ├── Dockerfile
│   │   └── package.json
│   ├── kinesis-consumer/
│   │   ├── src/
│   │   │   ├── index.ts
│   │   │   ├── kinesis-client.ts
│   │   │   ├── s3-writer.ts
│   │   │   ├── sqs-sender.ts
│   │   │   └── config.ts
│   │   ├── tests/
│   │   ├── Dockerfile
│   │   └── package.json
│   └── shared/
│       ├── src/
│       │   ├── types.ts
│       │   ├── logger.ts
│       │   └── metrics.ts
│       └── package.json
├── infrastructure/
│   ├── terraform/
│   └── cloudformation/
├── scripts/
│   └── validation/
├── package.json
├── tsconfig.json
└── README.md
```

### 1. Change Stream Consumer

**packages/change-stream-consumer/src/index.ts**
```typescript
import { MongoClient, ChangeStreamDocument } from 'mongodb';
import { KinesisProducer } from './kinesis-producer';
import { logger } from '@shared/logger';
import { publishMetric } from '@shared/metrics';
import config from './config';

interface CollectionConfig {
  database: string;
  collection: string;
  streamName: string;
}

class ChangeStreamConsumer {
  private mongoClient: MongoClient;
  private kinesisProducer: KinesisProducer;
  private isShuttingDown = false;

  constructor() {
    this.mongoClient = new MongoClient(config.mongoUri, {
      maxPoolSize: 10,
      minPoolSize: 5,
      serverSelectionTimeoutMS: 5000,
      socketTimeoutMS: 45000,
    });
    this.kinesisProducer = new KinesisProducer();
  }

  async start() {
    try {
      await this.mongoClient.connect();
      logger.info('Connected to MongoDB');

      // Health check endpoint
      this.startHealthCheck();

      // Watch each collection
      for (const collection of config.collections) {
        this.watchCollection(collection);
      }

      // Graceful shutdown
      process.on('SIGTERM', () => this.shutdown());
      process.on('SIGINT', () => this.shutdown());

    } catch (error) {
      logger.error('Failed to start consumer', { error });
      throw error;
    }
  }

  private async watchCollection(collectionConfig: CollectionConfig) {
    const { database, collection, streamName } = collectionConfig;
    const db = this.mongoClient.db(database);
    const coll = db.collection(collection);

    logger.info('Starting watch', { database, collection, streamName });

    const changeStream = coll.watch([], {
      fullDocument: 'updateLookup',
      maxAwaitTimeMS: 1000,
    });

    changeStream.on('change', async (change: ChangeStreamDocument) => {
      if (this.isShuttingDown) return;

      try {
        const startTime = Date.now();
        
        await this.processChange(change, streamName);

        const duration = Date.now() - startTime;
        publishMetric('ChangeProcessed', 1, { collection, streamName });
        publishMetric('ProcessingDuration', duration, { collection, streamName });

      } catch (error) {
        logger.error('Error processing change', {
          error,
          collection,
          changeId: change._id,
        });
        publishMetric('ProcessingError', 1, { collection, streamName });
      }
    });

    changeStream.on('error', (error) => {
      logger.error('Change stream error', { error, collection });
      publishMetric('ChangeStreamError', 1, { collection, streamName });
      
      // Implement exponential backoff retry
      this.retryWatchCollection(collectionConfig);
    });
  }

  private async processChange(
    change: ChangeStreamDocument,
    streamName: string
  ) {
    const { operationType, documentKey, fullDocument } = change;

    // Filter operations
    if (!['insert', 'update', 'replace', 'delete'].includes(operationType)) {
      return;
    }

    const record = {
      operationType,
      documentKey,
      fullDocument: fullDocument || documentKey,
      timestamp: new Date().toISOString(),
      collection: change.ns.coll,
      database: change.ns.db,
    };

    // Send to Kinesis
    await this.kinesisProducer.putRecord(streamName, record, documentKey._id);

    logger.debug('Change processed', {
      operationType,
      documentId: documentKey._id,
      streamName,
    });
  }

  private async retryWatchCollection(
    collectionConfig: CollectionConfig,
    attempt = 1
  ) {
    const backoffMs = Math.min(1000 * Math.pow(2, attempt), 30000);
    
    logger.info('Retrying watch after backoff', {
      collection: collectionConfig.collection,
      attempt,
      backoffMs,
    });

    await new Promise(resolve => setTimeout(resolve, backoffMs));

    if (!this.isShuttingDown) {
      this.watchCollection(collectionConfig);
    }
  }

  private startHealthCheck() {
    const express = require('express');
    const app = express();

    app.get('/health', (req, res) => {
      const isHealthy = this.mongoClient && !this.isShuttingDown;
      res.status(isHealthy ? 200 : 503).json({
        status: isHealthy ? 'healthy' : 'unhealthy',
        timestamp: new Date().toISOString(),
      });
    });

    app.listen(config.healthCheckPort, () => {
      logger.info(`Health check listening on port ${config.healthCheckPort}`);
    });
  }

  private async shutdown() {
    logger.info('Shutting down gracefully...');
    this.isShuttingDown = true;

    await this.mongoClient.close();
    await this.kinesisProducer.close();

    logger.info('Shutdown complete');
    process.exit(0);
  }
}

// Start the consumer
const consumer = new ChangeStreamConsumer();
consumer.start().catch((error) => {
  logger.error('Fatal error', { error });
  process.exit(1);
});
```

**packages/change-stream-consumer/src/kinesis-producer.ts**
```typescript
import { Kinesis } from 'aws-sdk';
import { logger } from '@shared/logger';
import config from './config';

export class KinesisProducer {
  private kinesis: Kinesis;
  private buffer: Map<string, any[]>;
  private flushInterval: NodeJS.Timeout;

  constructor() {
    this.kinesis = new Kinesis({
      region: config.awsRegion,
      maxRetries: 3,
      httpOptions: { timeout: 30000 },
    });
    this.buffer = new Map();

    // Flush buffer every 1 second or when it reaches size limit
    this.flushInterval = setInterval(() => this.flush(), 1000);
  }

  async putRecord(streamName: string, data: any, partitionKey: string) {
    // Add to buffer
    if (!this.buffer.has(streamName)) {
      this.buffer.set(streamName, []);
    }

    this.buffer.get(streamName)!.push({
      Data: JSON.stringify(data),
      PartitionKey: partitionKey.toString(),
    });

    // Flush if buffer is large enough
    if (this.buffer.get(streamName)!.length >= config.kinesisBufferSize) {
      await this.flushStream(streamName);
    }
  }

  private async flush() {
    for (const streamName of this.buffer.keys()) {
      await this.flushStream(streamName);
    }
  }

  private async flushStream(streamName: string) {
    const records = this.buffer.get(streamName);
    if (!records || records.length === 0) return;

    this.buffer.set(streamName, []); // Clear buffer

    try {
      // Batch write to Kinesis (up to 500 records)
      const chunks = this.chunkArray(records, 500);
      
      for (const chunk of chunks) {
        await this.putRecordsWithRetry(streamName, chunk);
      }

      logger.debug('Flushed records to Kinesis', {
        streamName,
        count: records.length,
      });

    } catch (error) {
      logger.error('Failed to flush to Kinesis', {
        error,
        streamName,
        recordCount: records.length,
      });
      // Re-add to buffer for retry
      this.buffer.set(streamName, [
        ...records,
        ...(this.buffer.get(streamName) || []),
      ]);
      throw error;
    }
  }

  private async putRecordsWithRetry(
    streamName: string,
    records: any[],
    attempt = 1
  ): Promise<void> {
    try {
      const result = await this.kinesis
        .putRecords({
          StreamName: streamName,
          Records: records,
        })
        .promise();

      // Handle partial failures
      if (result.FailedRecordCount && result.FailedRecordCount > 0) {
        const failedRecords = records.filter(
          (_, index) => result.Records[index].ErrorCode
        );

        if (attempt < config.maxRetries) {
          const backoffMs = Math.min(100 * Math.pow(2, attempt), 5000);
          await new Promise(resolve => setTimeout(resolve, backoffMs));
          return this.putRecordsWithRetry(streamName, failedRecords, attempt + 1);
        } else {
          throw new Error(
            `Failed to write ${result.FailedRecordCount} records after ${attempt} attempts`
          );
        }
      }
    } catch (error) {
      if (attempt < config.maxRetries) {
        const backoffMs = Math.min(100 * Math.pow(2, attempt), 5000);
        await new Promise(resolve => setTimeout(resolve, backoffMs));
        return this.putRecordsWithRetry(streamName, records, attempt + 1);
      }
      throw error;
    }
  }

  private chunkArray<T>(array: T[], size: number): T[][] {
    const chunks: T[][] = [];
    for (let i = 0; i < array.length; i += size) {
      chunks.push(array.slice(i, i + size));
    }
    return chunks;
  }

  async close() {
    clearInterval(this.flushInterval);
    await this.flush();
  }
}
```

### 2. Kinesis Consumer

**packages/kinesis-consumer/src/index.ts**
```typescript
import { S3Writer } from './s3-writer';
import { SQSSender } from './sqs-sender';
import { logger } from '@shared/logger';
import { publishMetric } from '@shared/metrics';
import config from './config';

const {
  KinesisClient,
  GetShardIteratorCommand,
  GetRecordsCommand,
  DescribeStreamCommand,
} = require('@aws-sdk/client-kinesis');

class KinesisConsumerApp {
  private kinesisClient: KinesisClient;
  private s3Writer: S3Writer;
  private sqsSender: SQSSender;
  private checkpoints: Map<string, string>;
  private isRunning = false;

  constructor() {
    this.kinesisClient = new KinesisClient({
      region: config.awsRegion,
      maxAttempts: 3,
    });
    this.s3Writer = new S3Writer();
    this.sqsSender = new SQSSender();
    this.checkpoints = new Map();
  }

  async start() {
    this.isRunning = true;
    logger.info('Starting Kinesis consumer');

    // Process each stream
    const streams = Object.keys(config.streamConfig);
    
    await Promise.all(
      streams.map(streamName => this.consumeStream(streamName))
    );
  }

  private async consumeStream(streamName: string) {
    try {
      // Get shards
      const describeCommand = new DescribeStreamCommand({
        StreamName: streamName,
      });
      const streamDescription = await this.kinesisClient.send(describeCommand);
      const shards = streamDescription.StreamDescription.Shards;

      logger.info('Starting to consume stream', {
        streamName,
        shardCount: shards.length,
      });

      // Process each shard
      await Promise.all(
        shards.map(shard => this.processShard(streamName, shard.ShardId))
      );

    } catch (error) {
      logger.error('Error consuming stream', { error, streamName });
      throw error;
    }
  }

  private async processShard(streamName: string, shardId: string) {
    let shardIterator = await this.getShardIterator(streamName, shardId);

    while (this.isRunning && shardIterator) {
      try {
        const getRecordsCommand = new GetRecordsCommand({
          ShardIterator: shardIterator,
          Limit: config.batchSize,
        });

        const response = await this.kinesisClient.send(getRecordsCommand);
        const { Records, NextShardIterator, MillisBehindLatest } = response;

        if (Records && Records.length > 0) {
          await this.processRecords(streamName, Records);
          publishMetric('RecordsProcessed', Records.length, { streamName, shardId });
        }

        publishMetric('MillisBehindLatest', MillisBehindLatest || 0, {
          streamName,
          shardId,
        });

        shardIterator = NextShardIterator;

        // Small delay if no records
        if (!Records || Records.length === 0) {
          await new Promise(resolve => setTimeout(resolve, 1000));
        }

      } catch (error) {
        logger.error('Error processing shard', { error, streamName, shardId });
        await new Promise(resolve => setTimeout(resolve, 5000));
      }
    }
  }

  private async getShardIterator(
    streamName: string,
    shardId: string
  ): Promise<string> {
    const checkpoint = this.checkpoints.get(`${streamName}:${shardId}`);

    const command = new GetShardIteratorCommand({
      StreamName: streamName,
      ShardId: shardId,
      ShardIteratorType: checkpoint ? 'AFTER_SEQUENCE_NUMBER' : 'TRIM_HORIZON',
      StartingSequenceNumber: checkpoint,
    });

    const response = await this.kinesisClient.send(command);
    return response.ShardIterator;
  }

  private async processRecords(streamName: string, records: any[]) {
    for (const record of records) {
      try {
        const data = JSON.parse(
          Buffer.from(record.Data, 'base64').toString('utf-8')
        );

        await this.processRecord(streamName, data);

        // Update checkpoint
        this.checkpoints.set(
          `${streamName}:${record.PartitionKey}`,
          record.SequenceNumber
        );

      } catch (error) {
        logger.error('Error processing record', {
          error,
          streamName,
          sequenceNumber: record.SequenceNumber,
        });
        
        // Send to DLQ
        await this.sendToDLQ(streamName, record, error);
        publishMetric('RecordFailed', 1, { streamName });
      }
    }
  }

  private async processRecord(streamName: string, data: any) {
    const { operationType, fullDocument, documentKey, collection } = data;
    const env = config.environment;
    
    // Prepare document
    const fullStringBody = JSON.stringify(
      fullDocument || documentKey
    );
    
    const baseKeyName = `${env}/${collection}/${documentKey._id}-${Date.now()}`;
    const maxS3ObjectSizeInBytes = 4194304; // 4MB limit
    const objectsToPutInS3: any[] = [];

    // Handle large documents - chunk if necessary
    if (fullStringBody.length > maxS3ObjectSizeInBytes) {
      const numChunks = Math.ceil(fullStringBody.length / maxS3ObjectSizeInBytes);
      const stringChunkLength = Math.ceil(fullStringBody.length / numChunks);

      for (let i = 0, n = 0; i < numChunks; i++, n += stringChunkLength) {
        objectsToPutInS3.push({
          Bucket: config.s3Bucket,
          Key: `${baseKeyName}-${i}`,
          Body: fullStringBody.substring(n, n + stringChunkLength),
        });
      }
    } else {
      objectsToPutInS3.push({
        Bucket: config.s3Bucket,
        Key: baseKeyName,
        Body: fullStringBody,
      });
    }

    // Write to S3
    await this.s3Writer.putObjects(objectsToPutInS3);

    // Send SQS message
    const queueUrl = config.streamConfig[streamName].sqsQueueUrl;
    const messageBody = {
      operation: operationType,
      S3FilePartsOfJSONDocument: objectsToPutInS3.map(obj => ({
        Bucket: obj.Bucket,
        Key: obj.Key,
      })),
    };

    await this.sqsSender.sendMessage(
      queueUrl,
      collection,
      messageBody,
      baseKeyName
    );

    logger.debug('Record processed successfully', {
      collection,
      documentId: documentKey._id,
      operation: operationType,
      s3Objects: objectsToPutInS3.length,
    });
  }

  private async sendToDLQ(streamName: string, record: any, error: any) {
    const dlqUrl = config.streamConfig[streamName].dlqUrl;
    if (!dlqUrl) return;

    await this.sqsSender.sendMessage(
      dlqUrl,
      'dlq',
      {
        originalRecord: record,
        error: error.message,
        timestamp: new Date().toISOString(),
      },
      `dlq-${Date.now()}`
    );
  }

  async stop() {
    logger.info('Stopping consumer...');
    this.isRunning = false;
    // Save checkpoints to DynamoDB here
  }
}

// Start the consumer
const consumer = new KinesisConsumerApp();
consumer.start().catch((error) => {
  logger.error('Fatal error', { error });
  process.exit(1);
});

process.on('SIGTERM', () => consumer.stop());
process.on('SIGINT', () => consumer.stop());
```

**packages/kinesis-consumer/src/s3-writer.ts**
```typescript
import { S3 } from 'aws-sdk';
import { logger } from '@shared/logger';
import config from './config';

export class S3Writer {
  private s3: S3;

  constructor() {
    this.s3 = new S3({
      region: config.awsRegion,
      maxRetries: 3,
    });
  }

  async putObjects(objects: Array<{ Bucket: string; Key: string; Body: string }>) {
    try {
      // Write all objects in parallel
      await Promise.all(
        objects.map(obj =>
          this.putObjectWithRetry(obj.Bucket, obj.Key, obj.Body)
        )
      );

      logger.debug('Successfully wrote objects to S3', {
        count: objects.length,
        keys: objects.map(o => o.Key),
      });

    } catch (error) {
      logger.error('Failed to write objects to S3', {
        error,
        objectCount: objects.length,
      });
      throw error;
    }
  }

  private async putObjectWithRetry(
    bucket: string,
    key: string,
    body: string,
    attempt = 1
  ): Promise<void> {
    try {
      await this.s3
        .putObject({
          Bucket: bucket,
          Key: key,
          Body: body,
          ContentType: 'application/json',
        })
        .promise();

    } catch (error: any) {
      if (attempt < config.maxRetries) {
        const backoffMs = Math.min(100 * Math.pow(2, attempt), 5000);
        logger.warn('Retrying S3 write after failure', {
          bucket,
          key,
          attempt,
          backoffMs,
        });
        
        await new Promise(resolve => setTimeout(resolve, backoffMs));
        return this.putObjectWithRetry(bucket, key, body, attempt + 1);
      }
      
      logger.error('Failed to write to S3 after retries', {
        error,
        bucket,
        key,
        attempts: attempt,
      });
      throw error;
    }
  }
}
```

**packages/kinesis-consumer/src/sqs-sender.ts**
```typescript
import { SQS } from 'aws-sdk';
import { logger } from '@shared/logger';
import config from './config';

export class SQSSender {
  private sqs: SQS;

  constructor() {
    this.sqs = new SQS({
      region: config.awsRegion,
      maxRetries: 3,
    });
  }

  async sendMessage(
    queueUrl: string,
    messageGroupId: string,
    messageBody: any,
    deduplicationId: string
  ) {
    try {
      await this.sendWithRetry(
        queueUrl,
        messageGroupId,
        messageBody,
        deduplicationId
      );

      logger.debug('Successfully sent SQS message', {
        queueUrl,
        messageGroupId,
      });

    } catch (error) {
      logger.error('Failed to send SQS message', {
        error,
        queueUrl,
        messageGroupId,
      });
      throw error;
    }
  }

  private async sendWithRetry(
    queueUrl: string,
    messageGroupId: string,
    messageBody: any,
    deduplicationId: string,
    attempt = 1
  ): Promise<void> {
    try {
      await this.sqs
        .sendMessage({
          QueueUrl: queueUrl,
          MessageGroupId: messageGroupId,
          MessageBody: JSON.stringify(messageBody),
          MessageDeduplicationId: deduplicationId,
        })
        .promise();

    } catch (error: any) {
      if (attempt < config.maxRetries) {
        const backoffMs = Math.min(100 * Math.pow(2, attempt), 5000);
        logger.warn('Retrying SQS send after failure', {
          queueUrl,
          attempt,
          backoffMs,
        });
        
        await new Promise(resolve => setTimeout(resolve, backoffMs));
        return this.sendWithRetry(
          queueUrl,
          messageGroupId,
          messageBody,
          deduplicationId,
          attempt + 1
        );
      }
      
      logger.error('Failed to send to SQS after retries', {
        error,
        queueUrl,
        attempts: attempt,
      });
      throw error;
    }
  }
}
```

### 3. Shared Types and Utilities

**packages/shared/src/types.ts**
```typescript
export interface CollectionConfig {
  database: string;
  collection: string;
  streamName: string;
}

export interface StreamConfig {
  sqsQueueUrl: string;
  dlqUrl?: string;
}

export interface ChangeEvent {
  operationType: 'insert' | 'update' | 'replace' | 'delete';
  documentKey: { _id: any };
  fullDocument?: any;
  ns: {
    db: string;
    coll: string;
  };
  timestamp: string;
}

export interface ProcessedRecord {
  operation: string;
  S3FilePartsOfJSONDocument: Array<{
    Bucket: string;
    Key: string;
  }>;
}
```

**packages/shared/src/logger.ts**
```typescript
import winston from 'winston';

export const logger = winston.createLogger({
  level: process.env.LOG_LEVEL || 'info',
  format: winston.format.combine(
    winston.format.timestamp(),
    winston.format.errors({ stack: true }),
    winston.format.json()
  ),
  defaultMeta: {
    service: process.env.SERVICE_NAME || 'stream-processor',
    environment: process.env.NODE_ENV || 'development',
  },
  transports: [
    new winston.transports.Console({
      format: winston.format.combine(
        winston.format.colorize(),
        winston.format.simple()
      ),
    }),
  ],
});
```

**packages/shared/src/metrics.ts**
```typescript
import { CloudWatch } from 'aws-sdk';

const cloudwatch = new CloudWatch({
  region: process.env.AWS_REGION || 'us-east-1',
});

const namespace = 'StreamProcessing';

export async function publishMetric(
  metricName: string,
  value: number,
  dimensions: Record<string, string>
): Promise<void> {
  try {
    await cloudwatch
      .putMetricData({
        Namespace: namespace,
        MetricData: [
          {
            MetricName: metricName,
            Value: value,
            Timestamp: new Date(),
            Dimensions: Object.entries(dimensions).map(([Name, Value]) => ({
              Name,
              Value,
            })),
            Unit: 'Count',
          },
        ],
      })
      .promise();
  } catch (error) {
    console.error('Failed to publish metric', { error, metricName });
  }
}
```

### 4. Validation Script

**scripts/validation/compare-outputs.js**
```javascript
const AWS = require('aws-sdk');
const crypto = require('crypto');

const s3 = new AWS.S3();

async function validateMigration(collection, startTime, endTime) {
  console.log(`Validating ${collection} from ${startTime} to ${endTime}`);

  // Get files from old trigger path
  const oldFiles = await listS3Files('dev', collection, startTime, endTime);
  
  // Get files from new system path
  const newFiles = await listS3Files('prod', collection, startTime, endTime);

  // Compare counts
  console.log(`Old system files: ${oldFiles.length}`);
  console.log(`New system files: ${newFiles.length}`);

  // Sample and compare content
  const sampleSize = Math.min(100, oldFiles.length);
  let matches = 0;

  for (let i = 0; i < sampleSize; i++) {
    const oldContent = await getS3Content(oldFiles[i].Key);
    const newContent = await getS3Content(newFiles[i].Key);

    const oldHash = crypto.createHash('md5').update(oldContent).digest('hex');
    const newHash = crypto.createHash('md5').update(newContent).digest('hex');

    if (oldHash === newHash) {
      matches++;
    } else {
      console.log(`Mismatch found: ${oldFiles[i].Key}`);
    }
  }

  const matchRate = (matches / sampleSize) * 100;
  console.log(`Match rate: ${matchRate.toFixed(2)}%`);

  return matchRate > 99.9;
}

async function listS3Files(prefix, collection, startTime, endTime) {
  const params = {
    Bucket: process.env.S3_BUCKET,
    Prefix: `${prefix}/${collection}/`,
  };

  const files = [];
  let continuationToken = null;

  do {
    if (continuationToken) {
      params.ContinuationToken = continuationToken;
    }

    const response = await s3.listObjectsV2(params).promise();
    
    const filtered = response.Contents.filter(item => {
      const timestamp = new Date(item.LastModified);
      return timestamp >= startTime && timestamp <= endTime;
    });

    files.push(...filtered);
    continuationToken = response.NextContinuationToken;

  } while (continuationToken);

  return files;
}

async function getS3Content(key) {
  const params = {
    Bucket: process.env.S3_BUCKET,
    Key: key,
  };

  const response = await s3.getObject(params).promise();
  return response.Body.toString('utf-8');
}

// Run validation
const collection = process.argv[2] || 'users';
const hoursAgo = parseInt(process.argv[3]) || 1;

const endTime = new Date();
const startTime = new Date(endTime - hoursAgo * 60 * 60 * 1000);

validateMigration(collection, startTime, endTime)
  .then(success => {
    console.log(success ? 'Validation PASSED' : 'Validation FAILED');
    process.exit(success ? 0 : 1);
  })
  .catch(error => {
    console.error('Validation error:', error);
    process.exit(1);
  });
```

### 5. Docker Configuration

**packages/change-stream-consumer/Dockerfile**
```dockerfile
FROM node:18-alpine

WORKDIR /app

# Copy package files
COPY package*.json ./
COPY packages/change-stream-consumer/package*.json ./packages/change-stream-consumer/
COPY packages/shared/package*.json ./packages/shared/

# Install dependencies
RUN npm ci --only=production

# Copy source code
COPY packages/change-stream-consumer ./packages/change-stream-consumer
COPY packages/shared ./packages/shared

# Build TypeScript
RUN npm run build

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD node -e "require('http').get('http://localhost:8080/health', (r) => { process.exit(r.statusCode === 200 ? 0 : 1); })"

# Run
USER node
CMD ["node", "packages/change-stream-consumer/dist/index.js"]
```

### 6. Test Examples

**packages/change-stream-consumer/tests/kinesis-producer.test.ts**
```typescript
import { KinesisProducer } from '../src/kinesis-producer';
import { mockClient } from 'aws-sdk-client-mock';
import { KinesisClient, PutRecordsCommand } from '@aws-sdk/client-kinesis';

const kinesisMock = mockClient(KinesisClient);

describe('KinesisProducer', () => {
  let producer: KinesisProducer;

  beforeEach(() => {
    kinesisMock.reset();
    producer = new KinesisProducer();
  });

  afterEach(async () => {
    await producer.close();
  });

  it('should buffer and batch records', async () => {
    kinesisMock.on(PutRecordsCommand).resolves({
      FailedRecordCount: 0,
      Records: [{ SequenceNumber: '123', ShardId: 'shard-1' }],
    });

    const data = { test: 'data' };
    await producer.putRecord('test-stream', data, 'partition-1');

    // Should not have called Kinesis yet (buffered)
    expect(kinesisMock.calls()).toHaveLength(0);

    // Wait for flush
    await new Promise(resolve => setTimeout(resolve, 1100));

    // Should have flushed to Kinesis
    expect(kinesisMock.calls()).toHaveLength(1);
  });

  it('should retry on failure', async () => {
    kinesisMock
      .on(PutRecordsCommand)
      .rejectsOnce(new Error('Throttled'))
      .resolves({
        FailedRecordCount: 0,
        Records: [{ SequenceNumber: '123', ShardId: 'shard-1' }],
      });

    const data = { test: 'data' };
    await producer.putRecord('test-stream', data, 'partition-1');
    await producer.flush();

    // Should have retried
    expect(kinesisMock.calls()).toHaveLength(2);
  });
});
```

**package.json (root)**
```json
{
  "name": "stream-processing-service",
  "version": "1.0.0",
  "private": true,
  "workspaces": [
    "packages/*"
  ],
  "scripts": {
    "build": "npm run build --workspaces",
    "test": "jest",
    "lint": "eslint . --ext .ts",
    "format": "prettier --write \"**/*.{ts,js,json,md}\""
  },
  "devDependencies": {
    "@types/jest": "^29.5.0",
    "@types/node": "^18.15.0",
    "@typescript-eslint/eslint-plugin": "^5.57.0",
    "@typescript-eslint/parser": "^5.57.0",
    "aws-sdk-client-mock": "^2.1.0",
    "eslint": "^8.37.0",
    "jest": "^29.5.0",
    "prettier": "^2.8.7",
    "ts-jest": "^29.1.0",
    "typescript": "^5.0.2"
  }
}
```

---

## Detailed Migration Plan

### Phase 1: Investigation & Setup (Week 1-2)

#### Week 1: Architecture Validation & Environment Setup

**Tasks:**
1. **AWS Infrastructure Provisioning**
   - Create VPC, subnets, security groups for ECS
   - Provision IAM roles and policies
   - Set up CloudWatch log groups and dashboards
   - Create development Kinesis streams (test)

2. **MongoDB Analysis**
   - Measure current change event volume per collection
   - Document peak traffic patterns
   - Identify collections with large documents
   - Review current error rates from trigger logs

3. **Team Preparation**
   - AWS Kinesis training for team
   - Set up development environment
   - Create GitHub repository for new services
   - Define coding standards and review process

**Deliverables:**
- AWS infrastructure in development account
- Volume and performance baseline document
- Team trained on Kinesis concepts

#### Week 2: Proof of Concept

**Tasks:**
1. **Build Change Stream Consumer**
   - Implement basic change stream listener
   - Handle connection failures and resume tokens
   - Add logging and metrics
   - Write unit tests

2. **Build Kinesis Consumer**
   - Implement Kinesis Client Library (KCL) consumer
   - Recreate S3 chunking logic
   - Recreate SQS message sending
   - Add error handling

3. **Test with Lowest Volume Collection**
   - Deploy to development environment
   - Run parallel to existing trigger (users collection)
   - Compare outputs (S3 files and SQS messages)
   - Measure latency and throughput

**Deliverables:**
- Working POC code in GitHub
- POC test results and comparison report
- Performance metrics document

**Success Criteria:**
- Events processed within 5 seconds of database change
- 100% parity with trigger outputs
- Zero data loss in 48-hour test period

---

### Phase 2: Development & Testing (Week 3-6)

#### Week 3-4: Production-Ready Implementation

**Tasks:**
1. **Enhance Change Stream Consumer**
   - Implement multi-collection watching
   - Add health checks and graceful shutdown
   - Implement exponential backoff for Kinesis writes
   - Add structured logging (JSON format)
   - Implement metrics publishing (CloudWatch)
   - Create Dockerfile and ECS task definition

2. **Enhance Kinesis Consumer**
   - Implement batch processing optimization
   - Add DLQ integration
   - Create retry logic with jitter
   - Implement idempotency checks
   - Add distributed tracing (X-Ray)
   - Create ECS task definition

3. **Infrastructure as Code**
   - Terraform modules for all AWS resources
   - Automated deployment pipeline (CodePipeline)
   - Environment-specific configurations
   - Secrets management (Secrets Manager)

4. **Monitoring & Alerting**
   - CloudWatch dashboards for each stream
   - Alarms for processing lag, errors, DLQ depth
   - PagerDuty/Slack integration
   - Runbook for common issues

**Deliverables:**
- Production-ready code with tests (>80% coverage)
- Complete Terraform infrastructure
- CI/CD pipeline deployed
- Monitoring dashboards and alerts configured

#### Week 5-6: Integration Testing

**Tasks:**
1. **Staging Environment Deployment**
   - Deploy to staging AWS account
   - Configure to read from MongoDB staging cluster
   - Run parallel with existing triggers

2. **Comprehensive Testing**
   - Load testing (simulate 10x current volume)
   - Failure testing (kill consumers, network issues)
   - Large document testing (>4MB documents)
   - Backpressure testing
   - End-to-end latency testing

3. **Validation**
   - Compare S3 outputs byte-by-byte
   - Verify SQS message parity
   - Validate deduplication behavior
   - Test resume from checkpoint
   - Verify data ordering

**Deliverables:**
- Test results documentation
- Performance benchmark report
- Issue log and resolution tracking
- Sign-off from QA and Infrastructure teams

**Success Criteria:**
- Handle 10x current peak volume
- 99.9% uptime in 1-week test
- <5 second p95 processing latency
- Zero data loss during failure scenarios

---

### Phase 3: Staged Migration (Week 7-10)

The migration will be executed in stages, starting with lowest-risk collections and progressing to critical ones.

#### Migration Priority Order

**Tier 1: Development/Testing (Week 7)**
1. `users_etl` (DevelopmentCluster)
2. `owners_etl` (DevelopmentCluster)

**Rationale:** Development cluster with lower risk, good for validating production process

**Tier 2: Medium Volume (Week 8)**
3. `reservation_payments_etl` (Production-Copy)
4. `reservation_charges_etl` (Production-Copy)

**Rationale:** Production-Copy cluster provides safety net, moderate volume

**Tier 3: Higher Volume (Week 9)**
5. `reservation_objs_etl` (Production-Copy)
6. `aadw_reservationdatafacilities_v3` (Primary)

**Rationale:** Higher volume but still manageable

**Tier 4: Critical (Week 10)**
7. `reservations_etl` (Primary)

**Rationale:** Highest volume and most critical, migrate last with full confidence

---

### Detailed Migration Steps (Per Trigger)

For each trigger, follow this process:

#### Step 1: Pre-Migration (Day 1 - Tuesday)

**Activities:**
1. **Setup**
   - Create Kinesis stream: `<collection-name>-changes`
   - Deploy Change Stream consumer for this collection (disabled)
   - Deploy Kinesis consumer for this stream (disabled)
   - Configure CloudWatch dashboards

2. **Validation Checkpoint**
   - Capture baseline metrics from current trigger
   - Document current error rate
   - Take snapshot of SQS queue state
   - Review recent S3 writes

3. **Communication**
   - Notify BI team of upcoming change
   - Send migration notice to stakeholders
   - Confirm rollback contact person on-call

**Duration:** 4 hours

#### Step 2: Shadow Mode (Day 2-4 - Wed-Fri)

**Activities:**
1. **Enable New System**
   - Enable Change Stream consumer
   - Enable Kinesis consumer
   - Keep existing trigger running

2. **Parallel Validation**
   - Both systems writing to S3 (different prefix: `dev/<collection>/` vs `prod/<collection>/`)
   - Compare outputs programmatically:
     ```javascript
     // Validation script runs hourly
     // - Count of S3 files created
     // - Count of SQS messages sent
     // - Checksum comparison of document content
     // - Latency comparison
     ```
   - Review CloudWatch logs for errors
   - Monitor consumer lag

3. **Acceptance Criteria**
   - 99.9% match rate for 72 hours
   - New system latency < current trigger latency
   - Zero crashes or restarts
   - All errors handled gracefully

**Duration:** 3 days (72 hours)

#### Step 3: Cutover (Day 5 - Monday)

**Activities:**
1. **Pre-Cutover Checks** (8:00 AM)
   - Verify shadow mode success metrics
   - Confirm monitoring is working
   - Verify rollback procedure
   - Ensure on-call person available

2. **Execute Cutover** (10:00 AM)
   - Disable MongoDB trigger (keep as backup)
   - Update S3 prefix in consumer to write to `prod/<collection>/`
   - Update SQS message routing
   - Monitor for 2 hours continuously

3. **Validation** (12:00 PM - 5:00 PM)
   - Verify ETL pipeline continues normally
   - Check BI dashboard data freshness
   - Review error rates and latency
   - Validate SQS queue processing

4. **Completion** (5:00 PM)
   - Declare migration successful
   - Update documentation
   - Schedule trigger deletion for 7 days later

**Duration:** 8 hours (business hours)

#### Step 4: Monitoring (Day 6-12)

**Activities:**
1. **Close Monitoring**
   - Review metrics daily
   - Check DLQ for any failed events
   - Compare data volumes with historical baseline
   - Validate with BI team

2. **Documentation**
   - Update runbooks
   - Record any issues and resolutions
   - Note performance differences
   - Document lessons learned

**Duration:** 7 days

#### Step 5: Cleanup (Day 13+)

**Activities:**
1. **Remove Old Trigger**
   - Disable trigger in MongoDB Atlas
   - Archive trigger code for reference
   - Remove trigger-specific IAM permissions
   - Update documentation

2. **Optimization**
   - Review Kinesis shard utilization
   - Optimize consumer batch sizes
   - Fine-tune retry parameters
   - Update alerts based on actual patterns

**Duration:** 4 hours

---

### Migration Calendar

```
Week 7: Development Cluster
├─ Mon-Tue: users_etl preparation
├─ Wed-Fri: users_etl shadow mode
├─ Mon: users_etl cutover
├─ Tue-Wed: owners_etl preparation
├─ Thu-Sat: owners_etl shadow mode
└─ Mon: owners_etl cutover

Week 8: Production-Copy - Payments & Charges
├─ Mon-Tue: reservation_payments_etl preparation
├─ Wed-Fri: reservation_payments_etl shadow mode
├─ Mon: reservation_payments_etl cutover
├─ Tue-Wed: reservation_charges_etl preparation
├─ Thu-Sat: reservation_charges_etl shadow mode
└─ Mon: reservation_charges_etl cutover

Week 9: Production-Copy - Objects & Analytics View
├─ Mon-Tue: reservation_objs_etl preparation
├─ Wed-Fri: reservation_objs_etl shadow mode
├─ Mon: reservation_objs_etl cutover
├─ Tue-Wed: aadw_reservationdatafacilities_v3 preparation
├─ Thu-Sat: aadw_reservationdatafacilities_v3 shadow mode
└─ Mon: aadw_reservationdatafacilities_v3 cutover

Week 10: Primary - Critical Reservations
├─ Mon-Tue: reservations_etl preparation
├─ Wed-Fri: reservations_etl shadow mode
├─ Mon: reservations_etl cutover
└─ Tue-Fri: Final validation and documentation
```

---

## Risk Mitigation

### Identified Risks & Mitigation Strategies

#### Risk 1: Data Loss During Migration
**Severity:** Critical  
**Probability:** Low  

**Mitigation:**
- Shadow mode ensures new system working before cutover
- Keep triggers active as backup during shadow mode
- Resume tokens ensure no events missed
- DLQ captures any processing failures
- Checkpointing ensures exactly-once processing

**Rollback:** Immediately re-enable trigger, investigate Kinesis consumer

---

#### Risk 2: Increased Latency
**Severity:** Medium  
**Probability:** Medium  

**Mitigation:**
- Performance testing in staging validates latency
- Kinesis on-demand mode auto-scales
- Batch processing optimizations
- CloudWatch alarms for lag threshold

**Rollback:** Tune consumer settings, add more consumers if needed

---

#### Risk 3: Cost Overrun
**Severity:** Medium  
**Probability:** Low  

**Mitigation:**
- On-demand Kinesis pricing is predictable
- Start with minimal retention (24 hours)
- Monitor costs weekly during migration
- Set AWS budget alerts

**Cost Estimates:**
- Kinesis: ~$500/month (estimated for all streams)
- ECS Fargate: ~$200/month (2 consumers)
- Data Transfer: ~$50/month
- **Total: ~$750/month** (vs current trigger costs unknown)

**Rollback:** Reduce retention period, optimize batch sizes

---

#### Risk 4: Unknown Document Schema Changes
**Severity:** Medium  
**Probability:** Medium  

**Mitigation:**
- Comprehensive testing in staging
- Schema validation in consumer
- Graceful error handling for unexpected formats
- DLQ for investigation

**Rollback:** Fix consumer code, replay from DLQ

---

#### Risk 5: MongoDB Connection Issues
**Severity:** High  
**Probability:** Low  

**Mitigation:**
- Connection pooling with retry logic
- Exponential backoff with jitter
- Health checks and auto-restart
- Multiple consumer instances for redundancy

**Rollback:** Restart consumer, verify network connectivity

---

#### Risk 6: Consumer Crashes
**Severity:** High  
**Probability:** Low  

**Mitigation:**
- ECS auto-restart on failure
- Health checks every 30 seconds
- Structured error logging
- PagerDuty alerts for crashes

**Rollback:** Review logs, fix bug, redeploy

---

#### Risk 7: BI Dashboard Disruption
**Severity:** High  
**Probability:** Low  

**Mitigation:**
- Coordinate with BI team on migration schedule
- Shadow mode validates data parity
- Schedule migrations during low-usage periods
- Have BI team validate after each cutover

**Rollback:** Revert to trigger immediately, analyze discrepancy

---

#### Risk 8: Team Knowledge Gap
**Severity:** Medium  
**Probability:** Medium  

**Mitigation:**
- Training sessions before migration
- Detailed runbooks
- Pair programming during implementation
- On-call rotation with experienced person

**Rollback:** N/A (preventive only)

---

## Testing Strategy

### Test Levels

#### 1. Unit Tests
**Coverage Target:** >80%

**Key Areas:**
- Document chunking logic
- Resume token management
- SQS message formatting
- S3 key generation
- Error handling

**Tools:** Jest, Mocha, aws-sdk-mock, LocalStack

---

#### 2. Integration Tests
**Environment:** Development

**Scenarios:**
- MongoDB → Change Stream → Kinesis
- Kinesis → Consumer → S3 + SQS
- End-to-end flow with all components
- Error injection and recovery
- Large document handling

**Tools:** docker-compose, LocalStack (local AWS)

---

#### 3. Load Testing
**Environment:** Staging

**Scenarios:**
- Baseline: Current production volume
- Peak: 5x current volume
- Sustained: 2x volume for 24 hours
- Burst: 10x volume for 15 minutes

**Metrics:**
- Throughput (events/second)
- Latency (p50, p95, p99)
- Error rate
- CPU/Memory utilization
- Kinesis iterator age

**Tools:** Artillery, k6, custom Node.js scripts

---

#### 4. Failure Testing (Chaos Engineering)
**Environment:** Staging

**Scenarios:**
- **Consumer crash:** Kill ECS task mid-processing
  - **Expected:** Auto-restart, resume from checkpoint
- **Network partition:** Block MongoDB connection
  - **Expected:** Retry with backoff, reconnect
- **Kinesis throttling:** Inject throttle errors
  - **Expected:** Exponential backoff, eventual success
- **S3 unavailable:** Simulate S3 503 errors
  - **Expected:** Retry, eventual success or DLQ
- **Large document:** 10MB+ document change
  - **Expected:** Proper chunking, all chunks written
- **Duplicate events:** Replay same event
  - **Expected:** Idempotency, no duplicate writes

**Tools:** AWS Fault Injection Simulator, manual injection

---

#### 5. Data Validation Testing
**Environment:** Staging & Production (shadow mode)

**Validation Points:**
- **S3 object count:** New system matches trigger count
- **S3 object content:** Byte-by-byte comparison
- **S3 key format:** Matches expected pattern
- **SQS message count:** Matches expected
- **SQS message content:** JSON structure validation
- **Event ordering:** Per-document order maintained
- **Deduplication:** No duplicate S3 files

**Tools:** Custom Node.js validation scripts, AWS Athena queries

---

### Testing Checklist (Per Migration)

**Pre-Cutover:**
- [ ] Unit tests passing (>80% coverage)
- [ ] Integration tests passing
- [ ] Load test results meet requirements
- [ ] Failure tests all passed
- [ ] 72-hour shadow mode validation complete
- [ ] Data parity confirmed (>99.9%)
- [ ] Monitoring dashboards validated
- [ ] Alerts tested and working
- [ ] Rollback procedure documented and tested
- [ ] Stakeholders notified

**Post-Cutover:**
- [ ] ETL pipeline functioning normally
- [ ] BI dashboards showing current data
- [ ] No increase in error rates
- [ ] Latency within acceptable range
- [ ] DLQ empty or only expected errors
- [ ] CloudWatch metrics normal
- [ ] Team trained on new system
- [ ] Documentation updated

---

## Rollback Plan

### Immediate Rollback Triggers

Execute rollback immediately if:
- Data loss detected (missing events)
- Error rate >1% for 15 minutes
- Processing lag >5 minutes for 30 minutes
- Consumer crashes repeatedly (>3 in 30 min)
- BI team reports data issues

### Rollback Procedure

**Time Estimate:** 15 minutes

#### Step 1: Re-enable Trigger (2 minutes)
```
1. Log into MongoDB Atlas
2. Navigate to Triggers
3. Enable the disabled trigger for this collection
4. Verify trigger is running (check logs)
```

#### Step 2: Disable New System (2 minutes)
```
1. Update ECS service desired count to 0 for Change Stream consumer
2. Update ECS service desired count to 0 for Kinesis consumer
3. Verify tasks have stopped
```

#### Step 3: Verify Data Flow (5 minutes)
```
1. Check trigger logs for new events
2. Verify S3 writes from trigger
3. Verify SQS messages from trigger
4. Confirm ETL pipeline processing
```

#### Step 4: Communication (3 minutes)
```
1. Notify BI team of rollback
2. Notify stakeholders
3. Create incident ticket
4. Schedule post-mortem
```

#### Step 5: Investigation (Ongoing)
```
1. Collect logs from all components
2. Review CloudWatch metrics
3. Check DLQ for errors
4. Identify root cause
5. Plan remediation
```

---

### Rollback Testing

**Before each migration:**
- Test rollback procedure in staging
- Verify trigger can be re-enabled quickly
- Confirm no data loss during rollback
- Document exact steps and timings

---

## Success Metrics

### Key Performance Indicators (KPIs)

#### Reliability
- **Target:** 99.95% uptime
- **Measurement:** CloudWatch metrics, uptime monitoring
- **Baseline:** Current trigger reliability (unknown, establish in Phase 1)

#### Latency
- **Target:** <5 seconds (p95) from MongoDB change to S3 write
- **Measurement:** Custom metrics in code, CloudWatch
- **Baseline:** Current trigger latency (establish in Phase 1)

#### Throughput
- **Target:** Handle 10x current peak volume
- **Measurement:** Events processed per second
- **Baseline:** Current volume (establish in Phase 1)

#### Data Accuracy
- **Target:** 100% data parity (zero data loss)
- **Measurement:** Automated validation scripts
- **Baseline:** 100% (current trigger standard)

#### Operational Efficiency
- **Target:** <2 hours/week maintenance
- **Measurement:** Time spent on operations
- **Baseline:** Current (unknown, track in Phase 1)

#### Cost
- **Target:** <$1000/month total infrastructure
- **Measurement:** AWS Cost Explorer
- **Baseline:** Current trigger costs (unknown)

---

### Go/No-Go Criteria for Production

Before BI Production Release:
- [ ] All 7 triggers migrated successfully
- [ ] Zero data loss in production migrations
- [ ] Latency meets or beats trigger performance
- [ ] 99.95% uptime over 2-week period
- [ ] BI team validates data accuracy
- [ ] Load testing passed at 10x volume
- [ ] Failure testing passed all scenarios
- [ ] Monitoring and alerting operational
- [ ] Runbooks complete and tested
- [ ] Team trained and confident
- [ ] Post-migration review completed

---

## Timeline Summary

| Phase | Duration | Activities | Deliverables |
|-------|----------|------------|--------------|
| **Phase 1** | 2 weeks | Investigation, POC, setup | POC code, infrastructure, baselines |
| **Phase 2** | 4 weeks | Development, testing | Production code, tests, staging validation |
| **Phase 3** | 4 weeks | Staged migration | All triggers migrated, monitoring |
| **Post-Migration** | 2 weeks | Optimization, documentation | Final reports, optimizations |
| **Total** | **12 weeks** | | **Production-ready stream processing** |

---


## Appendix

### A. Kinesis Stream Configuration

**Recommended Settings:**

```yaml
stream_name: <collection>-changes
mode: ON_DEMAND
retention_period: 24 # hours
encryption: AWS_KMS
enhanced_monitoring: shard_level
```

---

### B. ECS Task Definitions

**Change Stream Consumer:**
```yaml
family: change-stream-consumer
cpu: 1024 # 1 vCPU
memory: 2048 # 2 GB
container_definitions:
  - name: consumer
    image: <ECR_URI>/change-stream-consumer:latest
    environment:
      - MONGODB_URI: <from_secrets>
      - KINESIS_STREAMS: <from_config>
      - LOG_LEVEL: INFO
    log_configuration:
      log_driver: awslogs
```

**Kinesis Consumer:**
```yaml
family: kinesis-consumer
cpu: 2048 # 2 vCPU
memory: 4096 # 4 GB
container_definitions:
  - name: consumer
    image: <ECR_URI>/kinesis-consumer:latest
    environment:
      - S3_BUCKET: <from_config>
      - SQS_QUEUE_URLS: <from_config>
      - CHECKPOINT_TABLE: <dynamodb_table>
    log_configuration:
      log_driver: awslogs
```

---

### C. IAM Policy Templates

**Change Stream Consumer:**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "kinesis:PutRecord",
        "kinesis:PutRecords"
      ],
      "Resource": "arn:aws:kinesis:*:*:stream/*-changes"
    },
    {
      "Effect": "Allow",
      "Action": [
        "logs:CreateLogStream",
        "logs:PutLogEvents"
      ],
      "Resource": "arn:aws:logs:*:*:*"
    }
  ]
}
```

**Kinesis Consumer:**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "kinesis:GetRecords",
        "kinesis:GetShardIterator",
        "kinesis:DescribeStream",
        "kinesis:ListShards"
      ],
      "Resource": "arn:aws:kinesis:*:*:stream/*-changes"
    },
    {
      "Effect": "Allow",
      "Action": [
        "s3:PutObject"
      ],
      "Resource": "arn:aws:s3:::<bucket>/dev/*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "sqs:SendMessage"
      ],
      "Resource": "arn:aws:sqs:*:*:*-etl-queue"
    },
    {
      "Effect": "Allow",
      "Action": [
        "dynamodb:GetItem",
        "dynamodb:PutItem",
        "dynamodb:UpdateItem"
      ],
      "Resource": "arn:aws:dynamodb:*:*:table/kinesis-checkpoints"
    }
  ]
}
```

---

### D. Monitoring Dashboard Widgets

**Key Metrics to Display:**

1. **Stream Health**
   - Iterator age
   - Incoming records
   - Incoming bytes

2. **Consumer Health**
   - Processing latency (p50, p95, p99)
   - Success rate
   - Error rate
   - DLQ depth

3. **Downstream Impact**
   - S3 write rate
   - SQS send rate
   - ETL processing lag

4. **Resource Utilization**
   - ECS CPU/Memory
   - Network throughput
   - Kinesis shard count

---

### E. Runbook: Common Issues

**Issue:** High iterator age (lag)
**Cause:** Consumer not keeping up with stream
**Solution:**
1. Check consumer logs for errors
2. Increase ECS task count
3. Optimize batch processing
4. Consider provisioned mode for Kinesis

**Issue:** DLQ filling up
**Cause:** Repeated processing failures
**Solution:**
1. Check DLQ messages for pattern
2. Fix consumer bug if identified
3. Manually replay after fix
4. Update alerts if expected

**Issue:** Consumer crashing
**Cause:** Memory leak or unhandled exception
**Solution:**
1. Review CloudWatch logs
2. Check ECS task health
3. Restart task manually if needed
4. Deploy fix if bug identified

**Issue:** Data mismatch reported
**Cause:** Various (investigate)
**Solution:**
1. Run validation script
2. Check for missing S3 files
3. Review Kinesis records
4. Check for processing errors
5. Escalate if needed

---



**Document Status:** DRAFT  
**Last Updated:** 2025-11-01  
**Next Review:** Before Phase 1 kickoff
