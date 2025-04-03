## KAFKA

Kafka is a Queue service or we can say it is Event Streaming service.
Kafka is mainly used for Massive data handling.

Kafka Server called Broker. Kafka Cluster can contains multiple Server/Brokers which can work together.

Kafka has conecpts like Topic, wy which we can send and receive messages to specific Queues. Here instead of Queues we called Topic. Each topic can have multiple partitions. Each partions contains Subset of messages of specific topic. Every Partions is Ordered and also messages within a partitoin are also ordered.

Partitions allow Kafka to scale Horizontally. Each partition can be used parralley by multiple machines.

Kafka hold messages even after the consumed, and multiple consumer can read same message at different time.

Multiple consumers togetherly can create Consumer Group. Kafka ensure that only consumer of the group consume each message. 

Every message in Kafka has an offset. Offset is like Page number of a book. it identifies the position of the messgae in the partition. Consumer remember last offset they processed, if consumer crashes it can continue from the last  offset so no message is missed or read twice.

Kafka follow Leader/Master Slave/Follower where Leader Server is handle read write for a partiton, followers server replicates data from leader, after reader creash then one follower is elected for leader, followers are inactive only job is copy data from leader.

Kafka is frequently used in microservices architectures for event-driven communication between services. Each microservice communicates with others asynchronously via Kafka topics.

In any use case, we only need to use two Kafka functions: one is Write (for producing messages) WriteMessages and the other is Reader (for consuming messages) ReadMessage.

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