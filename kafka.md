## KAFKA
Apache Kafka is an open-source distributed event streaming platform used to handle real-time data feeds. It’s designed to handle massive amounts of data at high throughput in a fault-tolerant way. 

A Broker is a Kafka server. Kafka cluster can consist of multiple brokers/servers working together.

Kafka works with `Topics` and `Partitions`. Topic is just a stream or channel where messages are sent to and read from. Topic is continuous flow of specific data. Every message that is written into the Topic is append to the stream. 

Each topic in Kafka is split into multiple partitions. A partition is a log file that holds a subset of messages within the topic. 

Every partition is ordered, so messages within a partition are guaranteed to be read in the order they were written. 

Partitions allow Kafka to scale horizontally. Each partition can be read or written to by different machines in parallel, improving throughput and fault tolerance. 

`Producers` write to Topics, and `Consumers` read from Topics, but Kafka retains messages for a configurable time (even after they are consumed). Kafka is often used in pub-sub patterns where multiple consumers can read the same message at different times.

A `Consumer Group` is a collection of consumers that work together to read data from a Kafka topic. Kafka ensures that each message is consumed by only one consumer within the group, providing parallelism without duplication. If a consumer fails, another consumer in the group takes over its task.

Imagine you're reading a book. The `offset` is like the page number you're on. Each time you read a new message from Kafka, the offset moves forward by 1 (like turning a page). The offset helps you remember where you left off so you can continue reading from the right spot next time. Every message in Kafka has an offset. It's a number that identifies the position of that message in the partition. Consumers keep track of which message they’ve read by remembering the last offset they processed. If a consumer crashes or restarts, it can continue from the last offset it was at, so no messages are missed or read twice.

Kafka uses `Zookeeper` for distributed coordination. Zookeeper is responsible for managing the Kafka cluster's metadata, leader election, and managing which broker holds which partition.

Each partition has one `leader` and zero or more `followers`. The Leader broker is the only broker that can handle all write and read requests for that partition. So, all producers (writing data) and consumers (reading data) interact with the Leader of a partition. The Follower brokers replicate the data from the Leader. They are passive and do not serve read or write requests directly. If the Leader fails, one of the followers is elected to become the new Leader, ensuring availability and fault tolerance.





