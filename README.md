# AWS Event-Driven Order Notification System

This serverless system uses AWS to handle order events, store them in a database, and process them via messaging services. It's designed using SNS, SQS, Lambda, and DynamoDB.

---

## ✅ Architecture Overview

### Components:
- **SNS**: Publishes new order notifications.
- **SQS**: Queues messages for processing.
- **Lambda**: Consumes from SQS and stores data.
- **DynamoDB**: Stores order info.
- **DLQ**: Captures messages that fail >3 times.

![Architecture Diagram](./architecture_diagram2.png)

---

## ⚙️ Setup Instructions

### Step 1: DynamoDB Table
- Name: `Orders`
- Partition key: `orderId` (String)

### Step 2: SNS Topic
- Name: `OrderTopic`

### Step 3: SQS Queues
- Create `OrderQueue` (Standard)
- Create `OrderDLQ`
- Link DLQ with `maxReceiveCount = 3`

### Step 4: Subscription
- Subscribe `OrderQueue` to `OrderTopic`

### Step 5: Lambda Function
- Name: `OrderProcessor`
- Runtime: **Node.js**
- IAM: Allow access to DynamoDB + SQS
- Trigger: `OrderQueue`
- Code:
```ts
import { DynamoDBClient } from "@aws-sdk/client-dynamodb"; 
import { DynamoDBDocumentClient, PutCommand } from "@aws-sdk/lib-dynamodb"; 

const client = new DynamoDBClient({}); 
const docClient = DynamoDBDocumentClient.from(client); 

export const handler = async (event) => { 
    for (const record of event.Records) { 
        try { 
            const snsMessage = JSON.parse(record.body); 
            const order = JSON.parse(snsMessage.Message); 
            console.log("✅ Parsed order:", order); 
            const params = { 
                TableName: "Orders", 
                Item: { 
                    orderId: order.orderId, 
                    userId: order.userId, 
                    itemName: order.itemName, 
                    quantity: order.quantity, 
                    status: order.status, 
                    timestamp: order.timestamp 
                } 
            }; 
            await docClient.send(new PutCommand(params)); 
            console.log(`🟢 Order saved: ${order.orderId}`); 
        } catch (err) { 
            console.error("❌ Failed to process order:", err); 
        } 
    } 
};
