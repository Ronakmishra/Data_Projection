# Project: Real-Time Data Streaming & CDC Pipeline with AWS

## 📺 Demo Video

[Watch the demo on YouTube](https://www.youtube.com/watch?v=sW-YuJ52iec)

## Overview

This project is a hands-on demo of how to build a **real-time, event-driven pipeline** using AWS — and more specifically, how to track changes using **Change Data Capture (CDC)** from DynamoDB in near real time.

We’re simulating something like a live sales or transactions system, where updates (new orders, modified entries, deleted items) need to be immediately captured, processed, and made available for analysis, _without waiting for a batch job to run overnight_.

This walkthrough is built around a **fully serverless** architecture - meaning no EC2s, no clusters to manage - just clean, modular AWS-native services working together.

---

## 🏗️ Architecture Diagram

![Architecture Diagram](./arch.png)

---

## What We're Solving

Traditional pipelines are great when you’re okay waiting hours for insights. But what if you're running a dashboard that needs to reflect _what just happened_? Like:

- A new sale was placed
- An existing record was updated
- A suspicious transaction was deleted

We want all of that to be:

- **Captured** instantly
- **Transformed** for downstream consumption
- **Stored** in a queryable format
- **Ready for analysis within seconds**

---

## The Flow — Start to Finish

### 1. Mock Sales Data into DynamoDB

A simple Python script generates sales data like `order_id`, `product_name`, `quantity`, and `price`, and inserts it into a DynamoDB table (`P1_OrdersRawTable`).

This acts like our operational DB — think of it as our production system.

### 2. CDC with DynamoDB Streams

Once you enable **DynamoDB Streams**, every change — insert, update, delete — triggers an event. That’s the basis for **Change Data Capture**.

We set the stream to `NEW_AND_OLD_IMAGES` so we can see both the new and previous versions of items when things are updated.

### 3. EventBridge Pipe to Kinesis

Instead of writing a consumer Lambda to read from the stream, we use **EventBridge Pipes** — a no-code way to route those CDC events into **Kinesis Data Stream**.

This is where the stream begins.

### 4. Kinesis Stream → Firehose → Lambda (Transform)

From here:

- **Kinesis Firehose** picks up records
- It invokes a **Lambda function** (in transform mode)
- That function does light transformations:

  - Extracts relevant fields
  - Adds a CDC tag (INSERT, MODIFY, REMOVE)
  - Converts timestamps into readable format

- The Lambda returns cleaned-up JSON, which Firehose then buffers

### 5. Firehose → S3 (Partitioned JSON)

Firehose writes those transformed records into S3 in near real-time. While not instant per record, it’s buffered and dumped in batches every few seconds.

We’ve enabled **dynamic partitioning** by date/hour, so files are neatly organized like:

```
/year=2025/month=08/day=01/hour=16/
```

This makes querying easier and faster later on.

### 6. Glue Crawler + Catalog Table

Once the files land in S3, **AWS Glue Crawler** scans the folder structure, infers the schema, and creates or updates a table in the **Glue Data Catalog**.

We can now treat this S3 folder like a real table.

### 7. Query with Athena

Using **Amazon Athena**, we query this "table" directly from S3 using SQL. Thanks to CDC, we can:

- Filter by `cdc_event_type` (e.g., only get INSERTs)
- Track when a change occurred with `creation_datetime`
- See the trail of changes to the same order over time

Sample query:

```sql
SELECT *
FROM p1_kinesis_firehose_destination
WHERE cdc_event_type = 'MODIFY'
ORDER BY creation_datetime DESC
LIMIT 10;
```

---

## AWS Services Involved

| AWS Service                | What it Does                                          |
| -------------------------- | ----------------------------------------------------- |
| **DynamoDB**               | Stores real-time mock sales records                   |
| **DynamoDB Streams**       | Emits change events (CDC) like INSERT/MODIFY/REMOVE   |
| **EventBridge Pipe**       | No-code stream router from DynamoDB Stream → Kinesis  |
| **Kinesis Data Stream**    | Buffers real-time records                             |
| **Kinesis Firehose**       | Transforms (via Lambda) and delivers to S3            |
| **AWS Lambda**             | Transforms CDC payloads (base64 decode, add metadata) |
| **S3**                     | Stores partitioned JSON data for querying             |
| **Glue Crawler + Catalog** | Creates table schema from JSON data                   |
| **Amazon Athena**          | Queries the data using SQL in real-time               |

---

## 💭 Real-Time CDC in Action

As soon as a change happens in DynamoDB — say a new order comes in — the pipeline triggers:

- That change is routed to Kinesis
- Firehose calls Lambda to transform it
- Transformed data lands in S3
- Glue makes it discoverable
- Athena lets you query it _within seconds_

You can even watch a live audit trail of changes using Athena.

---

## More Sample Queries

```sql
-- See recent updates only
SELECT *
FROM p1_kinesis_firehose_destination
WHERE cdc_event_type = 'MODIFY'
ORDER BY creation_datetime DESC
LIMIT 10;

-- Count inserts today
SELECT COUNT(*)
FROM p1_kinesis_firehose_destination
WHERE cdc_event_type = 'INSERT'
  AND date(creation_datetime) = current_date;

-- High value orders in last 10 minutes
SELECT *
FROM p1_kinesis_firehose_destination
WHERE price > 300
  AND creation_datetime >= now() - interval '10' minute;
```

---

## Future Enhancements

- Add SNS alerts for specific triggers (e.g., price > \$1000)
- Build a QuickSight dashboard
- Add version tracking or `OldImage` support for updates

---

## Final Thoughts

What we’ve built here is a **real-time streaming + CDC pipeline** that:

- Requires no servers or batch jobs
- Uses fully managed AWS services
- Lets you track changes, enrich them, and analyze in seconds

This is the kind of architecture modern teams are adopting — and hopefully this gave you a clear, hands-on look at how to build one from scratch!
