# Systems Design interview

**Problem statement,** how to count video view at large scale?

Interviewer: Let's design a system that:

1. Counts YouTubr video views
1. Counts likes in Facebook/Instagram
1. Calculates application perfromance metrics
1. Analyse data in real-time

**Question to ask as an interviewee**:

1. What does data analysis mean?
1. Who sends that data?
1. Who uses the results of this analysis?
1. What does real-time really mean?

**Why requirements clarification is so important?**

> Interviewer wants to understand how the candidate deals with ambiguity

> Why? Because this quality is so important for the daily job

Sofware Engineer:

> There could be many solution to the problem asked. And, only when I fully understand what features of the system we need to design, I can come up with a proper set of technologies and building blocks

Different engineers have different experiences and will use different stacks to count views

1. SQL Database (MySQL, PostgresSQL)
1. NoSQL Database (Cassandra, DynamoDB)
1. Cache (Redis)
1. Stream processing (Kafka + Spark)
1. Cloud native stream processing (Kinesis)
1. Batch processing (Hadoop MapReduce)

> Each has it's own pros and cons. Only pick those that address the system requirements. You may want to ask more questions, for example, look at can use the following categories: users & customers, scale, performance, cost.

1. Users/Customers

   1. Who will use the system?

      > all youtube users, wants to see only total views?

      > the owner of the videos, wants to see per hour stats of how fast their video is growing

      > a machine learning model, wants to use data to generate recommendations

   1. How the system will be used?
      > is it marketing department, to generate monthly reports?

1. Scale (read and write)
   1. How many read queries per second?
   1. How much data is queried per request?
   1. How many video views are processed per second by the system?
   1. Can there be spikes in traffic, and how big they may be?
1. Performance
   1. What is the expected write-to-read data delay?
      > can counts happen an hour later so we can we batch process or do we need real-time data analysis required on the fly
   1. What is expected p99 latency for read queries?
1. Should the design minimize the cost of development?
1. Should the design minimize the cost of maintenance?

> Spend a lot of time asking for these requirements. You would rather spend more time trying to find the scope of the question than jumping to solve a more complicated problem that is no where close to the solution.

## 1. Functional requirements - API

What things the system has to do, the system has to count video view events so:

1. `countViewEvent(videoId)`
1. `countEvent(videoId, eventType)`
   > where event type parameter could be of **views, likes, shares**
1. `processEvent(videoId, eventType, function)`
   > where function is of **count, sum, average**
1. `processEvents(listOfEvents)`
   > generalize, by allowing system to not process one by one but can batch a list (containing an object with details of requests) of events and process them

What data can be retirved? ex. the system has to return video view count for a **time period**:

1. `getViewsCount(videoId, startTime, endTime)`
1. `getCount(videoId, eventType, startTime, endTime)`
1. `getStats(videoId, eventType, function, startTime, endTime)`

> We can use functional requirements to create our api and generalize it it's usage and make several iterations to generalize the api

## 2. Non-Functional requirements

> The interviewer might challenge us with questions ex. Let's design a system that can handle YouTube scale & let's try to make it as fast as possible

Sofware Developer:

> CAP Theorem tells me I should be choosing between availability and consistencu, I will go with availability

1. Scalable
   > Tens of thousands of video views per second
1. Highly Performant
   > few tens of milliseconds to return total views count for a video
1. Highly Available
   > Survives hardware/networkk failures, no single point of failure

## 3. High-level architechture

**Let's start with something simple**

user <- browser <- processing service <- database -> query service -> browser -> user

Interview:

> I have so many questions, where should I start?

Software Developer:

> I should betterbe driving this conversation. Otherwise, I may quickly become lost in questions. And the first thing I want to do is understand what data we need to store and how we do it

1. We need to define a data model, what we store

   1. Individual events e.g every click with other details like timestamp, country
      > Pros: Fast writes, can slice and dice data however we need, can recalculate numbers if needed
      > Cons: Slow reads, Costly for a large scale eg. many events
   1. Aggregate data e.g per minute, in real-time
      > Pros: Fast reads
      > Cons: Requires data aggregation pipeline eg. very hard to fix bugs when they happen

   > It is hard to pick one. So ask the interviewer what is important. It might be better to use both. However the draw back of using both is that it will be costly but highly beneficials.

Interviwer:

> Can you please give me a specific database name and explain your choice?

Software developer:

> I know that both SQL and NoSQL databases can scale and perform well. Let me evaluate both types.

The evaluation:

> How to **scale writes**?

> How to **scale reads**?

> How to make bith **writes** and **reads fast**?

> How **not to lose data** in case of hardware faults and network partitions?

> How to achieve **strong consistency**?

> What are the tradeoffs?

> How to **recover data** in case of outage?

> How to unsure **data security**?

> How to make it **extensible** for data model changes in the future?

> Where to run **cloud** vs **on-promises** data centers?

> How much money will it all **cost**?

**SQL database, MySQL**

(Processing service, Query service) -> cluster proxy -> (shard proxy -> MySQL shardA-M, shard proxy -> MySQL shardN-Z) -> (Read Replica incase of outage)

> this is similar to what youtube uses for system design

![Youtube system design](./resources/system-design.png)

> Using no-sql example with cassandra

![System two - NoSQL database](./resources/no-sql-database.png)

> In Functional requirements we chose availability over consistency - we prefer to show users stale data than no data at all. Cassandra extends the concept of eventual consistency by offering tunable consistensy.

**How we store**

We want to build a report that shows the following properties.

> Informational about video

> No. of total views per hour for last several hours

> Information about the channel that the video belongs to

### In SQL relational data bases, MySQL

**video_info**

| videoId | name | ...               | channelId |
| ------- | ---- | ----------------- | --------- |
| A       | name | Distributed cache | channelId |

**video_stats**

| videoId | timestamp | count |
| ------- | --------- | ----- |
| A       | 15:00     | 2     |
| A       | 16:00     | 3     |
| A       | 17:00     | 8     |

**channel_info**

| channelId | name                    | ... |
| --------- | ----------------------- | --- |
| 111       | System Design Interview | ..  |

> To generate the report mentioned above, we run a join query that retrieves data from all three tabels. An important property of a relational database is that data is normalized. This simply means we minimize data deplication across different tables eg. we only store video nams on the `video_info` table and do not store it on any other table, because if the video name changes the we have to change in all other tables which can lead to inconsistent data

> Relational system no longer think in terms of nouns, but in terms of **queries** that will be executed in the system we desire, normalization is quite normal

### in NoSQL database (cassandra, logical view)

> Here for same same data above, we strore every thing together. Instead of adding rows like in relational database, we add more columns for every hour

| videoId | cahnnel name            | video name        | 1:00 | 16:00 | 17:00 | ... |
| ------- | ----------------------- | ----------------- | ---- | ----- | ----- | --- |
| A       | System Design interview | Distributed Cache | 2    | 3     | 8     | ... |

There are 4 types of NoSQL databases

1. Column eg. HBase, Cassandra,
1. Document eg. mongodb
1. Key-Value eg. Rocksdb
1. Graph eg. Neo4j, AWS Neptune, TigerGraph

> We chose cassandra for our representation of a NoSQL database because it is:

1. Fault tolerant
1. Scalable, both read-write through put increases linearly as new machines are added
1. It supports multi data center replication
1. Works well with time series data

> Remember other NoSQL databases have different architecture and are not like cassandra which is a wide column database that supports asynchronous masterless replication

## 3. Processing service

> Basically, while we get video events we increase view counters

Start with the requirements.

1. How to **scale**?
1. How to **achieve high throughput**?
1. How **not to lose data** when processing node crashes?
1. What to do when **database** is **unavailable** or slow?

Sofware Developer:

> How can I make data processing scalable, reliable and fast?

> Hmmm... Scalability = Partitioning, Reliabile = Replication and checkpointing, Fast = In-memory. Let me see how I can combine all this.

### data aggregation basics

Should we pre-aggregate data in the processing service? Which option is better:

1. Event comes and we increment the counter right away
   ![Option 1: processing service](./resources/processing-service-1.png)

1. Event come and we accumulate the data in memory for each counter for a period in time then every several seconds we send that value to the database

![Option 2: processing service](./resources/processing-service-2.png)

### Push or Pull?

1. Push means some other service sends events synchronously to the processing service
1. Pull means the processing service pulls events from some temporary storage

> Although, the answer is that both options are totally valid, and we can make both work, the pull option has more advantages since it provides a better fault tolerance support and is easier to scale

Here is an example:

> If users are pushing events directly to the procession service, if the machine crashes the data will not be sent to the database and be lost. On the otherhand, we can have the processing service pull events from a temporary storage, if the machine fails, we still have data in our temporary storage

![push or pull](./resources/push-or-pull-benefits.png)

**Checkpointing**

Events arrive we put them in storage one by one, in order. Fixed order allows us to assign an offset for each event in the storage. Events are consumed sequentially. While when are progressing we write checkpoint to some persistent storage. If the machine crashes, we can replace it. The new machine will resume where the last machine failed

![check pointing](./resources/checkpointing.png)

> This is a useful concept in stream data processing

**Partitioning**

> Main idea remains the same when applied to event processing

1. Instead of putting all the events into a single queue, let's have several queues
1. Each queue is independent from the others
1. Every queue physically lives in it's own machine, and stores a subset of all events ex. we can get a hash of the videoId and use it this hash number to pick a queue

> Partitining allows use to parellalize events processing. More events we get, more partitions we create

![partitioning](./resources/partitioning.png)

### Processing service (detailed design)

1. Processing service reads events from partition one by one, counts it in memory and flushes this values to a database periodically
1. We need a component that reads events, an infinite loop that pulls data from the partition
   1. Partion consumer - It reads events and deserializes
      > Converts byte array into the actual object, usually this consumer component is a single threaded component. Meaning that we have a single thread that reads events. We can implement multi-threaded when several threads read from the partition in parallel. But this comes at a cost. Checkpointing becomes more complicated and It is hard to preserve the order of events.
      > Pros: Helps to eliminate duplicate events, we can use a distributed cache that stores the unique event identifiers for example the last 10 minutes, it the same messages arrive in the last 10 minute intervals the we only process one of them
   1. Aggregator
      1. In-memory store
   1. Internal queue
   1. Database writer
      1. Embeded Database
      1. Dead-letter queue

![detailed processing service](./resources/detail-processing-service.png)

### Data engestion path

![data ingestion path](./resources/data-ingestion-path.png)

### Ingestion path components

1. Partitioner Service Client

   1. Blocking vs Non-blocking I/O
   1. Buffering and batching
   1. Timeouts
   1. Retries
   1. Exponential backoff and jitter
   1. Circuit breaker pattern

      > Drawbacks (trade-offs): makes the system hard to test.

1. Load Balancer

   > Distributes data traffic between multiple servers. There hardware and software load balancers. e.g Elastic Load Balancer from AWS

   1. Software vs Hardware load balancer
   1. Networking protocols e.g

      > a.) http load balancer - can look inside the message and make a load balancing decision based on the content of the message e.g cookie or head

      > b.) tcp balancer - forwards network packets without inspecting it

   1. Load balancing algorithms e.g

      > round-robin algorithm

      > least connetion algorithm

      > least reponse time algorithm

      > hash based algorithms

      > DNS load balancing

1. Partitioner Service Partitions

   1. Partitions Strategy
   1. Service discovery
   1. Replication

      > Single-leader replication

      > Leaderless replication

      > Multileader replication

   1. Message format
      > Text
      > Binary format - faster to parse, shema is important
      > json format - increases message size, because it adds labeling of data
      > protocol buffer

### Data retrieval path

user -> Browser -> api gateway -> query service -> database (hot storage, cold storage, distributed cache)

### Data flow simulation

users -> api gateway -> load balancer -> partitioner service -> partitions -> partition consumer -> aggregator -> queue -> database writer -> database -> distrubuted cache -> query service -> user

### Technology stack

1. Client-side: Netty, Netflix Hystrix, Polly
1. Load balancing: NetScaler, NGINX, AWS ELB
1. Messaging Systems: Apache Kafka, AWS Kinesis
1. Data processing: Apache, Spark, Apache Flink, AWS Kinesis Data Analysis
1. Storage: Apache Cassandra, Apache HBase, Influx DB, Apache Hadoop, AWS Redshift, AWS S3

**Other**

1. Database solution (servers youtube database traffic since 2007): Vitess
1. Distributed cache: Redis
1. Deadletter Q mechanism: AWS SQS/ RabbitMQ
1. Data retrievement: RocksDB
1. Leader election for partitions and server discovery: Apache Zookeeper
1. Server discovery: Neflix Eureka
1. Monitoring our servie: AWS CloudWatch/ELK

> Binary: Avro, protobuf

> MurmurHash hashing function while partitioning

### Bottlenecks, tradeoffs

1. How to identify bottlenecks

1. How do you make sure the system is running healthy?

   > metrics

1. How to make sure the system produces accurate results?

   > Audit system. Make sure the expects results are what is the expected. You can use a Lambda System that does both a streaming and batching system and compare.

1. Let's say we've got a hugely popular video. Will it become a bottlenck?

1. What if the processing service cannot keep up with the speed new messages arrive?

## Summary

Functional requirements (API) -> non-functional requirements (qualities, sucure) -> detailed design -> bottlenecks and tradeoffs

**Functional requirements**

1. write down verbs
1. Define input parameters and return values
1. Make several iterations

**non-functional requirements (qualities)**

1. scalibily
1. Availability
1. Performance

**high-level design**

1. How data gets in
1. How data gets out
1. Try to drive the conversation

**Detailed design**

1. It's all about data (storage, transfer, processing)
1. Use fundamental concepts
1. Apply relevant technologies

**Bottlenecks and tradeoffs**

1.  Listen to the interviewer
