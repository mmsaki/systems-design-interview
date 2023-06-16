# Systems Design interview

**Problem statement,** how to count video view at large scale?

Interviewer: Let's design a system that:

1. Counts YouTubr video views
1. Counts likes in Facebook/Instagram
1. Calculates application perfromance metrics
1. Analyse data in realtime

**Question to as an interviewee**:

1. What does data analysis mean?
1. Who sends that data?
1. Who uses the results of this analysis?
1. What does real time really mean?

**Why requirements clarification is so important**

> Interviewer: I want to understand how the candidate deals with ambiguity

> Interviewer: Why? Because this quality is so important for the daily job

Sofware Engineer:

> There could be many solution to the problem asked. And, only when we fully understand what features of the system we need to design, we can come up with a proper set of technologies and building blocks.

Different engineer with different experiences will use count views

1. SQL Database (MySQL, PostgresSQL)
1. NoSQL Database (Cassandra, DynamoDB)
1. Cache (Redis)
1. Stream processing (Kafka + Spark)
1. Cloud native stream processing (Kinesis)
1. Batch processing (Hdoop MapReduce)

> Each has it's own pros and cons. Only pick those that are address the requirements. You may want to ask more questions. For example, we can use the following categories:

> Users/Customers, Scale (read/write), Performance, Cost

1. Users/Customers
   1. Who will use the system?
      > _a.) all youtube users, wants to see total views only?, b.) the owner of the videos, wants to per hour stats, c.) a machine learning model, wants to use data to generate recommendations_
   1. How the sysem will be used?
      > _a.) marketing department, to generate monthly reports_
1. Scale (read and write)
   1. How many read queries per second?
   1. How much data is queried per request?
   1. How many video views are processed per second by the system?
   1. Can there be spikes in traffic, and big they may be?
1. Performance
   1. What is expected write-to-read data delay? can counts happen an hour later, can we batch process or is realtime data analysis required
   1. What is expected p99 latency for read queries?
1. Should the design minimize the cost of development?
1. Should the design minimize the cost of maintenance?

> Spend a lot of time asking for these requirements. You would rather spend more time trying to find that scope of the question that jumping to solve a more complicated problem that is no where close to the solution being asked.

## 1. Functional requirements - API

Things the system has to do, the system has to count video view events so:

- `countViewEvent(videoId)`
- `countEvent(videoId, eventType)`
  > where event type parameter could be of **views, likes, shares**
- `processEvent(videoId, eventType, function)`
  > where function is of **count, sum, average**
- `processEvents(listOfEvents)`
  > generalize, by allowing system to not process one by one but can batch a list (containing an object with details of requests) of events and process them

Data reviews, the system has to return video view count for a **time period**:

- `getViewsCount(videoId, startTime, endTime)`
- `getCount(videoId, eventType, startTime, endTime)`
- `getStats(videoId, eventType, function, startTime, endTime)`

> This allows use to create our api and generalize it

## Ingestion path components

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

      > a.) round-robin algorithm

      > b.) least connetion algorithm

      > c.) least reponse time algorithm

      > c.) hash based algorithms

      > d.) DNS load balancing

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

# Data retrieval path

user -> Browser -> api gateway -> query service -> database (hot storage, cold storage, distributed cache)

# Data flow simulation

users -> api gateway -> load balancer -> partitioner service -> partitions -> partition consumer -> aggregator -> queue -> database writer -> database -> distrubuted cache -> query service -> user

# Technology stack

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

# Bottlenecks, tradeoffs

1. How to identify bottlenecks

1. How do you make sure the system is running healthy?

   > metrics

1. How to make sure the system produces accurate results?

   > Audit system. Make sure the expects results are what is the expected. You can use a Lambda System that does both a streaming and batching system and compare.

1. Let's say we've got a hugely popular video. Will it become a bottlenck?

1. What if the processing service cannot keep up with the speed new messages arrive?

# Summary

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
