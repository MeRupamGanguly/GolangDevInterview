# GolangDevInterview

## REDIS

```bash
docker network create archnet
docker run -d --name redis --network=archnet -p 6379:6379 redis
```

Redis is InMemory Data Store. REDIS can be use as CACHE, PUB-SUB system, RATE-LIMITING mechanism, Key-Expiration system, Event-Driven System using same PUB-SUB mechanism, Distributed Locking.

REDIS Client:-
```go
import "github.com/redis/go-redis/v9"
----------------------------------------
rdb := redis.NewClient(&redis.Options{
		Addr: "redis:6379",
	})
```
### Caching
Caching stores the result of previous requests to avoid making the same network call repeatedly. Caching can happen at various levels, such as on the client, server, or even the CPU level.

In this context, we have two main storage systems: Cache and Database (DB). To keep them in sync, there are different strategies:

Write-Through Cache: when we write data, we update both the cache and the database at the same time. This ensures that both sources of truth (cache and DB) are always synchronized.

Write-Back Cache: when we write data, we update only the cache first and return the response immediately. The database is updated asynchronously, typically after a set time interval or under certain conditions. This improves performance but introduces a slight delay of syncing db and redis.

```go
// Example usage: Fetch data with ID 1
cacheKey := fmt.Sprintf("user:%d", 1)
cachedData, err := rdb.Get(ctx, cacheKey).Result()
if err == redis.Nil {
    // Cache miss, fetch data from the database
    var name string
    err = db.QueryRow("SELECT name FROM users WHERE id = ?", id).Scan(&name)
    if err != nil {
        log.Fatalf("Error querying database: %v", err)
    }else{
        // Store the fetched data in Redis with a 10-minute expiration
        err := rdb.Set(ctx, cacheKey, name, 10*time.Minute).Err()
        if err != nil {
			log.Printf("Error storing data in cache: %v", err)
		}
    }
}else {
	// Cache hit, return data
	fmt.Println("Cache hit, returning from Redis...")
	fmt.Printf("Fetched from Redis: %s\n", cachedData)
}
```

Cache Eviction Policy:  
When Redis is full, it needs a strategy to decide which keys to remove

You can set the eviction policy in Redis by using the CONFIG SET command.

CLI COMMANDS:-
```bash
# Set max memory limit
CONFIG SET maxmemory 2gb
# Set eviction policy to allkeys-lru
CONFIG SET maxmemory-policy allkeys-lru
```

Redis provides the following eviction policies:

`noeviction`: No keys are evicted when memory is full, and Redis returns an error if new keys are added.

`volatile-lru`: Evicts the least recently used (LRU) keys only with an expiration time set.It doesn't evict keys without an expiration time.

`allkeys-lru`: Evicts the least recently used (LRU) keys regardless of expiration time.

`volatile-ttl`: Evicts keys with the shortest TTL (expiration time) first.

`allkeys-random`: Evicts random keys, regardless of whether they have an expiration time.

`volatile-random`: Evicts random keys with an expiration time set.

`volatile-lfu`: Evicts the least frequently used (LFU) keys only with an expiration time.

`allkeys-lfu`: Evicts the least frequently used (LFU) keys, regardless of expiration time.

```go
// Setting eviction policy via Redis config (Redis should be configured with maxmemory-policy)
rdb := redis.NewClient(&redis.Options{
    Addr: "localhost:6379",
})
err := rdb.ConfigSet(ctx, "maxmemory-policy", "allkeys-lru").Err()
if err != nil {
    fmt.Println("Error setting eviction policy:", err)
}
```
### PUB-SUB/Event
Redis Sends message using PUBLISH command, Subscriber listen to the Same Channel using SUBSCRIBE command.

```go
// Publisher
rdb.Publish(ctx, "order_channel", "New order placed!")
```
```go
// Subscriber
pubsub := rdb.Subscribe(ctx, "order_channel")
ch := pubsub.Channel()
for msg := range ch {
    fmt.Println("Received:", msg.Payload)
}
```
### Rate Limiting
Rate-Limiting control the number of request a user can make in a specific time window.
Redis track the IP and Request counts. After a certain number of request with a specified period the IP is blocked for sometime.

```go
func checkRateLimit(ip string, limit int, window int) bool {
    key := fmt.Sprintf("rate_limit:%s", ip)
    count, err := rdb.Incr(ctx, key).Result()
    if err != nil {
        log.Fatal(err)
    }
    if count == 1 {
        rdb.Expire(ctx, key, time.Duration(window)*time.Second)
    }
    return count <= int64(limit)
}
```
### Expiration Time
Redis allows setting TTL (Time-To-Live) for keys, automatically deleting them after a specified duration. This is useful for caching, sessions, and temporary tokens.

```go
err := rdb.SetEX(ctx, "verification_code", "123456", 5*time.Minute).Err()
if err != nil {
    log.Fatal(err)
}
```
### Distributes Caching
Imagine you have an e-commerce platform with limited stock for an item. When multiple users are trying to purchase the last available item simultaneously, you want to ensure that only one of them succeeds in making the purchase while others are blocked from trying to purchase the same item.

Redis supports distributed locks to ensure only one process accesses a resource at a time. This is useful in preventing race conditions in distributed systems.

SETNX sets a key only if it doesn't already exist, ensuring mutual exclusion. If the lock is already held by another user, it will return false,

Once the operation is complete, the lock is released by deleting the key.

```go
var rdb *redis.Client
var ctx = context.Background()

// Simulating an e-commerce purchase for the last item
func attemptPurchase(userID string) {
    lockKey := "purchase_lock:product_123" // Unique lock for the product
    // Attempt to acquire the lock
    // UserID as the value to identify who holds the lock and Lock timeout to avoid deadlock
    success, err := rdb.SetNX(ctx, lockKey, userID, 30 * time.Second).Result() 
    if err != nil {
        log.Fatal(err)
    }
    if !success {
        fmt.Println("Another user is already processing this purchase.")
        return // If lock acquisition failed, exit the function
    }
    // Sleep to Simulating the purchase process (e.g., reduce stock, charge payment)
    time.Sleep(5 * time.Second) 
    // Complete the purchase and release the lock
    err = rdb.Del(ctx, lockKey).Err()
    if err != nil {
        log.Fatal(err)
    }
    fmt.Printf("User %s successfully purchased the item!\n", userID)
}

func main() {
    // Initialize Redis client
    rdb = redis.NewClient(&redis.Options{
        Addr: "localhost:6379", 
    })
    // Simulate multiple users trying to purchase the last item
    go attemptPurchase("user_1")
    go attemptPurchase("user_2")
    // Wait for the goroutines to finish
    time.Sleep(10 * time.Second)
}
```


