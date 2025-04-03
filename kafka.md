## KAFKA

Kafka is an event streaming service (often referred to as a distributed event streaming platform) rather than a typical queue service. Kafka is primarily used for handling massive amounts of data in real-time.

In Kafka, the server is called a Broker. A Kafka Cluster consists of multiple brokers that work together to handle the data flow efficiently.

Kafka uses topics to send and receive messages, which can be thought of as analogous to queues in traditional messaging systems. However, in Kafka, we call them topics. Each topic can have multiple partitions. Each partition contains a subset of the messages for a specific topic. The messages within a partition are ordered, and each partition maintains its own order of messages.

Partitions allow Kafka to scale horizontally. This means that each partition can be processed in parallel by multiple machines, making Kafka capable of handling large-scale data streams.

Kafka retains messages even after they are consumed, and multiple consumers can read the same message at different times.

Multiple consumers can form a consumer group. Kafka ensures that only one consumer from the group consumes each message within a topic partition. This allows for load balancing and fault tolerance within the consumer group.

Each message in Kafka has an offset. The offset is like the page number in a book; it uniquely identifies the position of a message within a partition. Consumers track the offset of the messages they have consumed. If a consumer crashes, it can resume from the last successfully processed offset, ensuring that no messages are missed or read twice.

Kafka follows a Leader-Follower model for partition replication. The leader broker is responsible for handling both read and write requests for a partition. Follower brokers replicate data from the leader. If the leader broker crashes, one of the follower brokers is elected to become the new leader, ensuring high availability and fault tolerance. Followers do not handle client requests directly; they simply replicate data from the leader.

Kafka is commonly used in microservices architectures for event-driven communication between services. Each microservice can communicate asynchronously with others via Kafka topics.

In most use cases, we primarily deal with two functions in Kafka: one is producing messages (writing messages), and the other is consuming messages (reading messages).

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

The producer will collect messages into batches before sending them to Kafka. This BatchSize setting defines the maximum size of a single batch.

This LingerMillis setting allows the producer to wait for additional messages to fill up the batch before sending it. If a batch hasn't been filled up to the configured BatchSize, the producer will wait for LingerMillis milliseconds before sending it, even if the batch size isn't fully reached.

For RequiredAcks Field we have 3 Options:
kafka.NoResponse: No acknowledgment (lowest latency, highest risk).

kafka.WaitForLocal: Acknowledge once the leader replica acknowledges.

kafka.RequireAll: Wait for acknowledgment from all replicas (highest durability).

IDempotent setting enables or disables idempotent producers. When idempotency is enabled, Kafka ensures that even if the producer sends the same message multiple times (due to retries), the message is only written once to the Kafka topic.

Transport is the TLS configuration. It defines how the connection will be secured with SSL/TLS.
NEVER use InsecureSkipVerify: true in a production environment because it makes your connection vulnerable to man-in-the-middle (MITM) attacks. Always set it to false in production.


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