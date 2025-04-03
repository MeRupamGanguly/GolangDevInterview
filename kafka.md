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



Kafka is frequently used in microservices architectures for event-driven communication between services. Each microservice communicates with others asynchronously via Kafka topics.

Kafka can be used to aggregate logs from multiple services into a central logging system, making it easy to monitor and analyze logs in real time.

In any use case, we only need to use two Kafka functions: one is Write (for producing messages) WriteMessages and the other is Reader (for consuming messages) ReadMessage.

The producer will collect messages into batches before sending them to Kafka. This BatchSize setting defines the maximum size of a single batch.

This setting allows the producer to wait for additional messages to fill up the batch before sending it. If a batch hasn't been filled up to the configured BatchSize, the producer will wait for LingerMillis milliseconds before sending it, even if the batch size isn't fully reached.

For RequiredAcks Field we have 3 Options:
kafka.NoResponse: No acknowledgment (lowest latency, highest risk).

kafka.WaitForLocal: Acknowledge once the leader replica acknowledges.

kafka.RequireAll: Wait for acknowledgment from all replicas (highest durability).

IDempotent setting enables or disables idempotent producers. When idempotency is enabled, Kafka ensures that even if the producer sends the same message multiple times (due to retries), the message is only written once to the Kafka topic.

Transport is the TLS configuration. It defines how the connection will be secured with SSL/TLS.
NEVER use InsecureSkipVerify: true in a production environment because it makes your connection vulnerable to man-in-the-middle (MITM) attacks. Always set it to false in production.

```go
// Producer with advanced configurations (batch, idempotency, and SSL encryption)
func Producer(){   
    writer:= kafka.NewWriter(kafka.WriteConfig{
        Brokers:     []string{"localhost:9092"},
		Topic:       "my-topic",
		BatchSize:   1024 * 1024, // 1MB batch size. 
		LingerMillis: 100,        // Wait for 100ms before sending the batch
		RequiredAcks: kafka.RequireAll, // Ensures replication
		IDempotent:  true,  // Enable idempotency for Exactly Once Semantics (EOS)
		Transport: &kafka.Transport{
			TLS: &tls.Config{
				InsecureSkipVerify: true, // For development only; verify in production
			},
		},
    })
    defer writer.Close()
    for i := 0; i < 5; i++ {
		err := writer.WriteMessages(context.Background(),
			kafka.Message{
				Key:   []byte("key1"),
				Value: []byte("Message with advanced producer settings " + string(i)),
			},
		)
		if err != nil {
			log.Fatalf("failed to write message: %v", err)
		}
		log.Println("Message written to Kafka.")
	}
}
```

```go
// Consumer with advanced configuration offset
reader := kafka.NewReader(kafka.ReaderConfig{
	Brokers: []string{"localhost:9092"},
	Topic:   "my-topic",
	GroupID: "my-consumer-group",
})
defer reader.Close()

for {
	msg, err := reader.ReadMessage(context.Background())
	if err != nil {
		log.Fatalf("failed to read message: %v", err)
	}
	log.Printf("Received message: %s", string(msg.Value))
    // Manually committing offset after processing
	err = reader.CommitMessages(context.Background(), msg)
	if err != nil {
		log.Fatalf("failed to commit offset: %v", err)
	}
	log.Printf("Manually committed offset for message: %s", string(msg.Value))
}
```