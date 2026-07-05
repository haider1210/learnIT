# Redis VS PostgresSQL
This is the **Docker Voting App** architecture (a very popular Docker sample project). The key confusion is why it uses **Redis** and **PostgreSQL (DB)** together instead of only one database.

Let's understand the flow step by step.

## Architecture

```text
           User
             |
      +-------------+
      | Vote (Python) |
      +-------------+
             |
             | Store vote
             v
         +--------+
         | Redis  |
         +--------+
             |
             | Read vote
             v
      +---------------+
      | Worker (.NET) |
      +---------------+
             |
             | Save permanently
             v
      +-------------+
      | PostgreSQL  |
      +-------------+
             |
             | Read results
             v
      +---------------+
      | Result (Node) |
      +---------------+
```

---

# What is Redis?

Redis is an **in-memory data store**.

* Stores data in **RAM**
* Extremely fast
* Used for

  * Cache
  * Queue
  * Session storage
  * Temporary data

Think of Redis as a **waiting room**.

```
Vote
 |
 v
Redis

Vote1
Vote2
Vote3
Vote4
```

It temporarily holds the votes.

---

# What is PostgreSQL (DB)?

PostgreSQL is a **relational database**.

* Stores data on disk
* Permanent storage
* Supports SQL
* ACID compliant
* Used for long-term data

Example table

| id | Vote |
| -- | ---- |
| 1  | Cats |
| 2  | Dogs |
| 3  | Cats |
| 4  | Cats |

This data remains even if the application restarts.

---

# Why not store directly into PostgreSQL?

Imagine **10,000 users** click Vote at exactly the same time.

Without Redis

```
User1 ---> PostgreSQL
User2 ---> PostgreSQL
User3 ---> PostgreSQL
User4 ---> PostgreSQL
...
10000 users
```

PostgreSQL receives thousands of writes simultaneously.

Possible problems:

* High latency
* Lock contention
* Database overload
* Slower response to users

---

With Redis

```
Users
   |
   |
Redis Queue
   |
   |
Worker
   |
PostgreSQL
```

The Vote application only needs to push votes into Redis, which is extremely fast.

The Worker processes them one by one (or in batches) and writes them to PostgreSQL.

---

# What is the Worker doing?

The Worker continuously checks Redis for new votes.

Pseudo-code:

```python
while True:
    vote = redis.pop()

    if vote:
        postgres.insert(vote)
```

So the Worker acts as a bridge between Redis and PostgreSQL.

---

# Purpose of Redis here

Redis is used as a **message queue**.

Example

```
Redis

--------------
Cat
Dog
Dog
Cat
Cat
--------------
```

Worker reads

```
Cat
```

Stores into PostgreSQL

Then reads

```
Dog
```

Stores again...

---

# Purpose of PostgreSQL

It stores the final permanent data.

```
Votes Table

---------------------
ID    Option
---------------------
1     Cat
2     Dog
3     Cat
4     Cat
---------------------
```

The Result application queries PostgreSQL to count votes:

```sql
SELECT option, COUNT(*)
FROM votes
GROUP BY option;
```

Output

```
Cats : 523

Dogs : 477
```

---

# Why does Result read PostgreSQL instead of Redis?

Redis only contains **new incoming votes** waiting to be processed. Once the Worker has moved a vote into PostgreSQL, it is removed from Redis.

Therefore, Redis does **not** hold the complete voting history.

PostgreSQL contains **all votes**, making it the correct source for calculating totals and displaying results.

---

# Real-world analogy

Imagine ordering food at a restaurant.

### Redis = Waiter

The waiter quickly takes your order.

```
Table 5

Pizza
```

The waiter doesn't cook it.

---

### Worker = Chef

The chef receives the order from the waiter.

```
Pizza
```

Starts preparing it.

---

### PostgreSQL = Restaurant Records

Once the pizza is prepared, the order is recorded permanently.

```
Order #1543

Pizza
Paid
Delivered
```

---

# Summary

| Component           | Purpose                                   | Speed             | Data Lifetime       |
| ------------------- | ----------------------------------------- | ----------------- | ------------------- |
| **Vote App**        | Receives user votes                       | Fast              | Temporary           |
| **Redis**           | Temporary in-memory queue for new votes   | Very Fast (RAM)   | Temporary           |
| **Worker**          | Reads votes from Redis and saves them     | Processing layer  | N/A                 |
| **PostgreSQL (DB)** | Permanent storage of all votes            | Slower than Redis | Persistent          |
| **Result App**      | Reads PostgreSQL and displays vote counts | Read-only         | Uses permanent data |

### Why use both?

Using Redis and PostgreSQL together combines the strengths of each:

* **Redis** handles bursts of incoming traffic with very low latency by acting as a temporary queue.
* **PostgreSQL** provides reliable, durable storage for all votes.

* https://www.youtube.com/watch?v=dmGW22W3VOs&list=PPSV
* The **Worker** decouples the front-end from the database, making the system more scalable and resilient. This pattern is common in production systems where fast ingestion and reliable persistence are both important.
