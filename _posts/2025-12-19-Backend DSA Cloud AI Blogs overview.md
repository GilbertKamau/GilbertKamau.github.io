---
title: Building a Scalable Backend with DynamoDB Streams — My Journey Before the AWS User Group Pwani Talk
date: 2025-12-06
categories: [Cloud]
tags: [aws, dynamodb, lambda, cloud, serverless]
comments: true
---

When I agreed to give an online presentation to the **AWS User Group Pwani**, I knew I didn’t want to show slides full of theory. I wanted something real — something I could **build, demo, and explain**.

So I decided to create a **small but scalable backend system** using:

- API Gateway  
- AWS Lambda  
- DynamoDB  
- DynamoDB Streams  

Everything worked perfectly during the live session (thank God 😅).  
But the journey of building it before the talk…  
that’s where the real learning happened.

This blog is about that journey — **what I built, what surprised me, and the gems I picked up along the way**.

Let’s dive in.

---

## 🌱 Why This Project?

Before the presentation, I wanted to answer one question:

> **How can you build a backend that can scale to millions of users without managing servers?**

AWS has many ways to answer that question, but **DynamoDB Streams** stood out because:

- They’re simple but powerful  
- They’re event-driven  
- They allow your backend to react to data in real time  

If you’ve ever wanted to track changes in your database — who registered, who updated their profile, who deleted their account — **Streams are the perfect tool**.

---

## 🧠 DynamoDB Streams — Explained Like You’re 10

Think of your DynamoDB table as a notebook.

Every time someone:

- adds a new line  
- edits a line  
- erases a line  

DynamoDB Streams captures that event like a **security camera**.

Then it whispers to your Lambda function:

- “Hey, someone added a new user!”  
- “Hey, someone updated their email!”  
- “Hey, someone deleted an item!”  

Your Lambda can react instantly.

That’s the magic of **event-driven architecture**.

---

## 🛠️ Step 1: Building the Backend API

I started with the essentials:

- ✔ API Gateway endpoint → `/register`  
- ✔ Lambda function → validates & stores user data  
- ✔ DynamoDB table → `Users`  
- ✔ Duplicate checks → no repeated emails or usernames  

When a user registers, the API saves:

```json
{
  "name": "Alice",
  "email": "alice@gmail.com",
  "username": "al"
}
```
and returns:
```
json
{
  "message": "User registered successfully"
}
```
Simple. Clean. Serverless.

🔄 Step 2: Turning on DynamoDB Streams
Next, I enabled Streams on the Users table with:

Stream type: NEW_AND_OLD_IMAGES

This means:

INSERT → record the new item

MODIFY → record both old and new images

DELETE → record the old image

Then I connected the Stream to a new Lambda:

📌 LogUserChangesFunction

This Lambda would “listen” to any change on the table.

⚙️ Step 3: Building the Stream Processor Lambda
Here’s the Lambda handler:

```
export const handler = async (event) => {
  console.log("=== DynamoDB Stream Event Received ===");

  for (const record of event.Records) {
    const type = record.eventName;

    try {
      if (type === "INSERT") {
        console.log("USER ADDED:", JSON.stringify(record.dynamodb.NewImage));
      } else if (type === "MODIFY") {
        console.log("USER UPDATED - OLD:", JSON.stringify(record.dynamodb.OldImage));
        console.log("USER UPDATED - NEW:", JSON.stringify(record.dynamodb.NewImage));
      } else if (type === "REMOVE") {
        console.log("USER DELETED:", JSON.stringify(record.dynamodb.OldImage));
      } else {
        console.log("UNKNOWN EVENT:", type, record);
      }
    } catch (err) {
      console.error("Error processing record:", err, record);
      throw err;
    }
  }

  return { statusCode: 200 };
};

```
This tiny function became my activity logger for the entire database.


🧪 Step 4: Testing — Where the Learning Really Happened
This is where the real story lives.

📌 Mistake 1: Expecting Streams to Show Old Data
I added a user before turning on Streams.

Later, when I checked the logs… nothing.

✅ Streams only capture events after they are enabled

❌ They do not show historical changes

📌 Mistake 2: Updating an Item Is Not an Insert
When I updated Eve’s username from toto → kichwa, I got:

```
USER UPDATED - OLD: ...
USER UPDATED - NEW: ...
Exactly what I wanted.
```

📌 Mistake 3: IAM Is the Silent Gatekeeper
At first, I got errors like:

“Cannot access stream. Ensure GetRecords, ListStreams permissions…”

Streams won’t even fire unless IAM is correct.

The Lambda needs:

```
dynamodb:GetRecords
dynamodb:GetShardIterator
dynamodb:DescribeStream
dynamodb:ListStreams
```

Once I fixed that, everything worked flawlessly.

📈 Why This Architecture Is Actually Scalable
This simple system becomes extremely powerful because each part scales automatically:

1️⃣ API Gateway
Handles thousands of requests per second.

2️⃣ Lambda
Scales by spawning new instances based on traffic.

3️⃣ DynamoDB
Scales horizontally and supports millions of reads/writes per second.

4️⃣ DynamoDB Streams
Process events asynchronously without slowing down the main API.

No servers.
No patching.
No capacity planning.

---
Just pure scalability.

🧩 What I Learned (The Real Takeaways)
✔ Serverless is not about writing less code

✔ Real scalability comes from loose coupling

✔ IAM is everything

✔ Logs are your best debugging tool

✔ Event-driven architecture is addictive

---

🎤 Final Thoughts
When I finally gave the presentation to AWS User Group Pwani, everything worked smoothly — no errors, no surprises, just a clean demo.

But the real win was everything I learned before that moment.

I didn’t just build an API.
I built a scalable, event-driven backend using real AWS production tools.

If you're curious about serverless, DynamoDB Streams is one of the best places to start.

Simple idea → big power.

And who knows — your next small experiment might become your next talk… or your next startup 🚀


