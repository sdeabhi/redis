# Redis Interview Questions and Answers

## Basic Questions

### 1. What is Redis, and what are its primary use cases?
**Answer**:  
Redis (Remote Dictionary Server) is an open-source, in-memory, key-value NoSQL database known for its high performance, low latency, and versatility. It supports various data structures like strings, lists, sets, hashes, and sorted sets, and is often used as a cache, message broker, or real-time database.  

**Primary Use Cases**:  
- **Caching**: Store frequently accessed data to reduce database load.  
- **Session Management**: Store user sessions for fast access.  
- **Pub/Sub Messaging**: Enable real-time notifications or event streaming.  
- **Task Queues**: Manage background jobs using lists.  
- **Analytics/Leaderboards**: Use sorted sets for ranking or counting.  
- **Geospatial Data**: Store and query location-based data.  

**Example**:  
Caching data in a Node.js app:  
```javascript
const cachedResult = await redisClient.get(`favorites:paginated:${userId || '*'}:${templeId || '*'}:${page}:${limit}:${sort}:${order}`);
if (cachedResult) return JSON.parse(cachedResult);
```

**Commands**: `SETEX`, `GET`, `DEL`, `HSET`, `PUBLISH`, `SUBSCRIBE`, `ZADD`.

---

### 2. What data structures does Redis support?
**Answer**:  
Redis supports:  
- **Strings**: Simple key-value pairs.  
- **Lists**: Ordered collections of strings.  
- **Sets**: Unordered collections of unique strings.  
- **Sorted Sets**: Sets with scores for ranking.  
- **Hashes**: Key-value pairs under a single key.  
- **Bitmaps**: For bit-level operations.  
- **HyperLogLog**: For counting unique items.  
- **Geospatial Indexes**: For location-based queries.  
- **Streams**: For log-like data processing.  

**Example**:  
Caching a favorite as a hash:  
```javascript
await redisClient.hSet(`favorite:${userId}:${templeId}`, favoriteData);
await redisClient.expire(`favorite:${userId}:${templeId}`, 3600);
```

**Commands**:  
- Strings: `SET`, `GET`, `INCR`.  
- Lists: `LPUSH`, `RPOP`, `LRANGE`.  
- Sets: `SADD`, `SMEMBERS`.  
- Sorted Sets: `ZADD`, `ZRANGE`.  
- Hashes: `HSET`, `HGETALL`.  
- Geospatial: `GEOADD`, `GEORADIUS`.  
- Streams: `XADD`, `XREAD`.

---

### 3. How does Redis differ from traditional databases like MongoDB?
**Answer**:  
- **Storage**: Redis is in-memory (fast, volatile) with optional persistence; MongoDB is disk-based (persistent, slower).  
- **Data Model**: Redis uses key-value with various structures; MongoDB uses JSON-like documents.  
- **Performance**: Redis offers sub-millisecond latency; MongoDB is slower but supports complex queries.  
- **Use Case**: Redis for caching, sessions, real-time tasks; MongoDB for persistent storage and queries.  
- **Scalability**: Redis is single-threaded with clustering; MongoDB supports sharding and replication.  

**Example**:  
Caching MongoDB results:  
```javascript
const cachedUsers = await redisClient.get('users');
if (cachedUsers) return JSON.parse(cachedUsers);
const users = await User.find().lean();
await redisClient.setEx('users', 3600, JSON.stringify(users));
```

**Commands**: `SETEX`, `GET`, `DEL`.

---

### 4. What are the advantages and disadvantages of Redis?
**Answer**:  
**Advantages**:  
- High performance due to in-memory storage.  
- Flexible data structures (strings, lists, sets, etc.).  
- Simple key-value model.  
- Scalable with replication and clustering.  
- Versatile for caching, pub/sub, queues.  

**Disadvantages**:  
- Memory-intensive for large datasets.  
- Limited persistence compared to disk-based databases.  
- Single-threaded processing.  
- No complex query support.  

---

### 5. What is the difference between Redis persistence options (RDB vs. AOF)?
**Answer**:  
- **RDB**: Snapshots dataset at intervals (e.g., every 60s if 1000 keys change). Compact, fast recovery, but risks data loss between snapshots.  
- **AOF**: Logs every write operation, replayed on restart. More durable, minimal data loss, but larger files and slower recovery.  

**Configuration Example**:  
```plaintext
save 60 1000  # RDB: Snapshot every 60s if 1000 keys change
appendonly yes  # Enable AOF
appendfsync everysec  # AOF: Sync every second
```

**Commands**: `BGSAVE`, `BGREWRITEAOF`.

---

## Intermediate Questions

### 6. How does Redis handle caching in a web application?
**Answer**:  
Redis caches frequently accessed data to reduce database load:  
1. Check cache (`GET`/`HGETALL`).  
2. On miss, query database, store result (`SETEX`/`HSET`) with TTL.  
3. On hit, return cached data.  
4. Invalidate cache (`DEL`) on data changes.  

**Example**:  
Caching paginated favorites:  
```javascript
const cacheKey = `favorites:paginated:${userId || '*'}:${templeId || '*'}:${page}:${limit}:${sort}:${order}`;
const cachedResult = await redisClient.get(cacheKey);
if (cachedResult) return JSON.parse(cachedResult);
const result = await FavoritesService.getPaginatedData(...);
await redisClient.setEx(cacheKey, 600, JSON.stringify(result));
```

**Commands**: `SETEX`, `GET`, `DEL`, `HSET`, `HGETALL`, `EXPIRE`.

---

### 7. Explain Redis Pub/Sub and how it’s used in real-time applications.
**Answer**:  
Redis Pub/Sub enables publishers to send messages to channels, and subscribers receive them. Ideal for real-time notifications. Messages are not persisted.  

**Example**:  
Publishing a notification:  
```javascript
await redisClient.publish('notifications', JSON.stringify({ templeId, message }));
redisClient.subscribe('notifications', (message) => {
  console.log('Notification:', JSON.parse(message));
});
```

**Commands**: `PUBLISH`, `SUBSCRIBE`, `PSUBSCRIBE`, `UNSUBSCRIBE`.

---

### 8. How do you implement rate limiting with Redis?
**Answer**:  
Use a token bucket algorithm:  
1. Track request count with `INCR`.  
2. Set TTL with `EXPIRE` for the time window.  
3. Check if count exceeds limit.  

**Example**:  
```javascript
const key = `rate:${req.ip}`;
const count = await redisClient.incr(key);
if (count === 1) await redisClient.expire(key, 60);
if (count > 10) return res.status(429).json({ message: 'Too many requests' });
```

**Commands**: `INCR`, `EXPIRE`, `TTL`.

---

### 9. What is Redis Cluster, and when would you use it?
**Answer**:  
Redis Cluster shards data across nodes, with masters and replicas for scalability and high availability. Use for large datasets, high availability, or horizontal scaling.  

**Example**:  
Distribute favorites keys across nodes.  

**Commands**: `CLUSTER INFO`, `CLUSTER NODES`.

---

### 10. How does Redis handle transactions?
**Answer**:  
Redis transactions use `MULTI`, `EXEC`, and `DISCARD` to queue and execute commands atomically. `WATCH` provides optimistic locking.  

**Example**:  
```javascript
const multi = redisClient.multi();
multi.incr(`user:${userId}:favorite_count`);
multi.setEx(`favorite:${userId}:${templeId}`, 3600, JSON.stringify(favorite));
await multi.exec();
```

**Commands**: `MULTI`, `EXEC`, `DISCARD`, `WATCH`, `UNWATCH`.

---

## Advanced/Tricky Questions

### 11. How do you handle Redis memory management in production?
**Answer**:  
- Set `maxmemory` (e.g., `maxmemory 2gb`).  
- Use `maxmemory-policy` (e.g., `volatile-lru`).  
- Monitor with `INFO MEMORY`.  
- Optimize data structures.  
- Enable persistence (RDB/AOF).  

**Example**:  
```bash
CONFIG SET maxmemory 2gb
CONFIG SET maxmemory-policy volatile-lru
```

**Commands**: `CONFIG SET`, `CONFIG GET`, `INFO MEMORY`.

---

### 12. What are Redis Streams, and how do they differ from Pub/Sub?
**Answer**:  
Streams are log-like structures for event data, with persistence, consumer groups, and replayability. Unlike Pub/Sub, messages are stored and can be read later.  

**Example**:  
```javascript
await redisClient.xAdd('notifications', '*', { templeId, message });
const messages = await redisClient.xRead([{ id: '0-0', stream: 'notifications' }], { count: 10 });
```

**Commands**: `XADD`, `XREAD`, `XGROUP`, `XACK`.

---

### 13. How would you implement a leaderboard using Redis?
**Answer**:  
Use sorted sets to store users with scores:  
1. Add users with `ZADD`.  
2. Retrieve top users with `ZRANGE`.  
3. Update scores with `ZINCRBY`.  

**Example**:  
```javascript
await redisClient.zAdd('leaderboard', [{ score: 10, value: userId }]);
const topUsers = await redisClient.zRange('leaderboard', 0, 9, { withScores: true });
await redisClient.zIncrBy('leaderboard', 1, userId);
```

**Commands**: `ZADD`, `ZRANGE`, `ZINCRBY`, `ZREM`.

---

### 14. What are Redis Lua scripts, and why are they useful?
**Answer**:  
Lua scripts execute atomically on the server, reducing round-trips. Useful for complex logic and performance.  

**Example**:  
```lua
local count = redis.call('GET', KEYS[1])
if count and tonumber(count) >= ARGV[1] then
  return 0
end
redis.call('INCR', KEYS[1])
redis.call('EXPIRE', KEYS[1], ARGV[2])
return 1
```
```javascript
const result = await redisClient.eval(script, { keys: [`favorite_count:${userId}`], arguments: ['10', '3600'] });
```

**Commands**: `EVAL`, `SCRIPT LOAD`, `EVALSHA`.

---

### 15. How do you secure a Redis instance in production?
**Answer**:  
- Set password with `requirepass`.  
- Enable TLS (`rediss://`).  
- Restrict access with firewall.  
- Disable dangerous commands (`rename-command`).  
- Bind to specific IPs (`bind 127.0.0.1`).  

**Example**:  
```plaintext
requirepass your_secure_password
bind 127.0.0.1
rename-command FLUSHALL ""
```

**Commands**: `CONFIG SET requirepass <password>`, `AUTH`.

---

## Project-Specific Questions

### 16. How did you integrate Redis into your FavoritesController?
**Answer**:  
Used Redis for caching:  
- **Individual Favorites**: `HSET`/`HGETALL` with keys like `favorite:<userId>:<templeId>`.  
- **Paginated Favorites**: `SETEX` with keys like `favorites:paginated:<userId>:<templeId>:<page>:<limit>:<sort>:<order>`.  
- **Analytics/Trends**: Cached with `SETEX`.  
- **Invalidation**: Cleared caches with `DEL` on create/delete/notify.  

**Example**:  
```javascript
const cacheKey = `favorite:${userId}:${templeId}`;
const cachedFavorite = await redisClient.hGetAll(cacheKey);
if (Object.keys(cachedFavorite).length > 0) return cachedFavorite;
const favorite = await FavoritesService.findOne({ userId, templeId });
await redisClient.hSet(cacheKey, favorite);
await redisClient.expire(cacheKey, 3600);
```

**Commands**: `HSET`, `HGETALL`, `SETEX`, `GET`, `DEL`, `EXPIRE`.

---

### 17. How do you handle cache invalidation in your project?
**Answer**:  
Invalidate caches on:  
- **createFavorite**: Clear individual, paginated, and analytics caches.  
- **deleteFavorite**: Same as above.  
- **notifyFavoriteUsers**: Clear paginated and analytics caches.  

**Example**:  
```javascript
await Promise.all([
  redisClient.del(`favorite:${userId}:${templeId}`),
  redisClient.del(`favorites:paginated:*:*`),
  redisClient.del(`favorites:analytics:*`),
]);
```

**Commands**: `DEL`.

---

### 18. How would you optimize the notifyFavoriteUsers function using Redis?
**Answer**:  
- Cache favorite users with `SETEX`.  
- Use Pub/Sub for real-time notifications.  
- Invalidate caches after notifications.  

**Example**:  
```javascript
const favoritesCacheKey = `favorites:paginated:*:${templeId}`;
const cachedFavorites = await redisClient.get(favoritesCacheKey);
let favorites;
if (cachedFavorites) {
  favorites = JSON.parse(cachedFavorites);
} else {
  favorites = await FavoritesService.getPaginatedData({ templeId }, "userId", {}, 1, 1000);
  await redisClient.setEx(favoritesCacheKey, 600, JSON.stringify(favorites));
}
await redisClient.publish('notifications', JSON.stringify({ templeId, message }));
```

**Commands**: `SETEX`, `GET`, `PUBLISH`.

---

## Additional Tricky Questions

### 19. What is the difference between `EXPIRE` and `SETEX`?
**Answer**:  
- **EXPIRE**: Sets TTL on an existing key.  
  ```bash
  SET mykey "value"
  EXPIRE mykey 60
  ```
- **SETEX**: Sets key, value, and TTL atomically.  
  ```bash
  SETEX mykey 60 "value"
  ```

---

### 20. How do you handle Redis replication lag in a high-traffic application?
**Answer**:  
- Monitor lag with `INFO REPLICATION`.  
- Add replicas or use Redis Cluster.  
- Read from replicas for non-critical data (`READONLY`).  
- Batch writes with pipelines or Lua scripts.  

**Example**:  
```bash
INFO REPLICATION
WAIT 1 1000
```