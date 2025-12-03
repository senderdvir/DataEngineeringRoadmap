IoT to Snowflake Data Pipeline

Process Flow Diagram

flowchart TD
    %% Styling Definitions
    classDef aws fill:#FF9900,stroke:#232F3E,stroke-width:2px,color:white;
    classDef snow fill:#29B5E8,stroke:#1A6B8F,stroke-width:2px,color:white;
    classDef edge fill:#555,stroke:#333,stroke-width:2px,color:white;
    classDef process fill:#66CC66,stroke:#333,stroke-width:1px,color:black;

    %% -------------------
    %% 1. Edge Layer
    %% -------------------
    subgraph Edge_Layer [Edge Devices]
        direction TB
        Robots[("🤖 Robots")]:::edge
    end

    %% -------------------
    %% 2. AWS Cloud
    %% -------------------
    subgraph AWS_Cloud [AWS Cloud]
        direction TB
        IoT[("AWS IoT Core\n(MQTT Broker)")]:::aws
        S3[("AWS S3 Bucket\n(Parquet Logs)")]:::aws
        SNS[("Amazon SNS")]:::aws
    end

    %% -------------------
    %% 3. Snowflake Data Cloud
    %% -------------------
    subgraph Snowflake [Snowflake Data Cloud]
        direction TB
        
        subgraph Ingest [Ingestion Layer]
            Snowpipe[("Snowpipe\n(Auto-Ingest)")]:::snow
            RawTable[("MRR.RAW_DATA\n(Variant/Raw)")]:::snow
        end

        subgraph Orchestration [Orchestration & Staging]
            Task5s(("Task\n(Every 5s)")):::process
            StgTable[("STG Tables")]:::snow
            Stream[("Snowflake\nStream")]:::snow
        end

        subgraph Transformation [Transformation Layer]
            TaskTransform(("Task\n(Consumer)")):::process
            SP[["Stored Procedure\n(Transform & Upsert)"]]:::process
            DWH[("DWH Tables\n(Final Analytics)")]:::snow
        end
    end

    %% -------------------
    %% Data Flow Connections
    %% -------------------
    
    %% Edge to Cloud
    Robots -->|"1. Publish Cycle Data"| IoT
    
    %% AWS Internal Flow
    IoT -->|"2. Buffering\n(x min or 128MB)"| S3
    S3 -.->|"3. Object Create Event"| SNS
    
    %% Cloud to Snowflake
    SNS -.->|"4. Trigger Notification"| Snowpipe
    S3 -->|"5. Load Parquet"| Snowpipe
    Snowpipe -->|"6. Copy Into"| RawTable
    
    %% Snowflake Internal Flow
    RawTable -->|"7. Select New Data"| Task5s
    Task5s -->|"8. Insert"| StgTable
    
    StgTable -.->|"9. Change Capture (CDC)"| Stream
    Stream -->|"10. Has Data?"| TaskTransform
    TaskTransform -->|"11. Execute"| SP
    SP -->|"12. Merge/Upsert"| DWH

    %% Descriptions for subgraphs
    style Edge_Layer fill:#f9f9f9,stroke:#333,stroke-dasharray: 5 5
    style AWS_Cloud fill:#fff4e5,stroke:#FF9900
    style Snowflake fill:#e0f7fa,stroke:#29B5E8


detailed Technical Flow

1. Ingestion (Edge & AWS)

Source: Robots publish telemetry messages (Clean Cycle notifications) to AWS IoT Core via MQTT.

Buffering: IoT Core batches these messages. The trigger condition for flushing to storage is a buffer limit of 128 MB or a time window of X minutes (whichever comes first).

Storage: The batched data is written to AWS S3 as Parquet files. Parquet is columnar and highly optimized for ingestion.

Event Notification: When a new Parquet file lands in S3, an S3 Event Notification triggers Amazon SNS.

2. Loading (Snowflake Raw)

Snowpipe: This is configured for Auto-Ingest. It listens to the SNS queue. When it receives the notification, it locates the new Parquet file in S3.

Target: Snowpipe executes a COPY INTO command automatically, loading the Parquet data into the MRR.RAW_DATA table. This table likely uses a VARIANT column to handle the schema evolution of the logs.

3. Orchestration & Transformation (Snowflake)

Micro-Batch Task (Task 1): * Schedule: Runs every 5 seconds.

Action: Reads from MRR.RAW_DATA, parses the structure if necessary, and inserts records into STG (Staging) tables.

Change Data Capture (Stream): A Snowflake Stream is attached to the STG table. It tracks the "delta" (offsets) of new rows inserted by the previous task.

Transformation Task (Task 2):

Trigger: This task monitors the Stream. It only runs if the Stream has data (SYSTEM$STREAM_HAS_DATA).

Action: It calls a Stored Procedure.

Final Processing (Stored Procedure):

The Stored Procedure (likely JavaScript, SQL, or Python) applies complex business logic, joins, and cleaning.

Upsert: It performs a MERGE operation (Update/Insert) into the final DWH tables to ensure data integrity and avoid duplicates.

Key Technologies

Technology

Role

Purpose

AWS IoT Core

Message Broker

Handles high-concurrency connectivity from robots.

AWS S3

Data Lake

Durable, cheap storage for raw log files (Parquet).

Amazon SNS

Event Bus

Low-latency notification to tell Snowflake data is ready.

Snowpipe

Ingestion Engine

Serverless loading of data as soon as it arrives.

Snowflake Tasks

Orchestration

Scheduling the 5-second micro-batches.

Snowflake Streams

CDC

efficiently tracking which rows need processing.

Stored Procedures

Transformation

Encapsulating the complex logic for the DWH merge.
