# Redis Interview Questions and Answers with Node.js

## Basic Questions

### 1. What is Redis, and how is it used in a Node.js application?
**Answer**:  
Redis is an in-memory, key-value NoSQL database known for its high performance and support for data structures like strings, lists, sets, hashes, and sorted sets. In Node.js applications, Redis is commonly used for caching, session management, real-time messaging, and task queues to enhance performance and scalability.  

**Use Cases in Node.js**:  
- **Caching**: Store API responses or database queries to reduce latency (e.g., MongoDB queries).  
- **Session Storage**: Manage user sessions with libraries like `connect-redis`.  
- **Pub/Sub**: Enable real-time notifications using Redis Pub/Sub.  
- **Rate Limiting**: Control API request rates.  

**Example**:  
Caching MongoDB data in a Node.js app using `node-redis`:  
```javascript
const redis = require('redis');
const client = redis.createClient({ url: process.env.REDIS_URL });
await client.connect();

const cacheKey = 'users';
const cached = await client.get(cacheKey);
if (cached) return JSON.parse(cached);
const users = await User.find().lean();
await client.setEx(cacheKey, 3600, JSON.stringify(users));
```

**Commands**: `SETEX`, `GET`, `DEL`.

---

### 2. How do you connect to Redis in a Node.js application?
**Answer**:  
In Node.js, the `node-redis` library is commonly used to connect to Redis. You configure the client with a Redis URL, handle connection events, and use async/await for operations.  

**Steps**:  
1. Install `node-redis`: `npm install redis`.  
2. Create a client with the Redis URL (from environment variables).  
3. Connect and handle errors/events.  

**Example**:  
```javascript
const redis = require('redis');
const client = redis.createClient({
  url: process.env.REDIS_URL || 'redis://localhost:6379',
});

client.on('error', (err) => console.error('Redis Error:', err));
client.on('connect', () => console.log('Connected to Redis'));

await client.connect();

// Example operation
await client.set('key', 'value');
```

**Commands**: `SET`, `GET`.  

**Tricky Note**: Always handle reconnection logic in production (e.g., exponential backoff) to manage network issues.

---

### 3. What Redis data structures are most useful in Node.js applications?
**Answer**:  
Redis supports multiple data structures, with the following being most useful in Node.js:  
- **Strings**: Cache JSON-serialized API responses or session data.  
- **Hashes**: Store structured objects (e.g., user profiles).  
- **Lists**: Implement task queues or message logs.  
- **Sets**: Track unique items (e.g., user IDs).  
- **Sorted Sets**: Create leaderboards or ranked feeds.  

**Example**:  
Store a user profile as a hash in Node.js:  
```javascript
await client.hSet('user:123', {
  username: 'john_doe',
  email: 'john@example.com',
});
const user = await client.hGetAll('user:123');
```

**Commands**:  
- Strings: `SET`, `GET`, `SETEX`.  
- Hashes: `HSET`, `HGETALL`.  
- Lists: `LPUSH`, `RPOP`.  
- Sets: `SADD`, `SMEMBERS`.  
- Sorted Sets: `ZADD`, `ZRANGE`.

---

## Intermediate Questions

### 4. How do you implement caching with Redis in a Node.js/Express API?
**Answer**:  
Caching in Node.js/Express with Redis reduces database load by storing frequently accessed data. The process involves checking the cache, querying the database on a miss, and storing the result with a TTL.  

**Steps**:  
1. Check Redis for cached data (`GET` or `HGETALL`).  
2. On cache miss, fetch from the database (e.g., MongoDB).  
3. Store result in Redis with `SETEX` or `HSET`.  
4. Invalidate cache (`DEL`) on data updates.  

**Example**:  
Cache a list of jobs in an Express route:  
```javascript
const express = require('express');
const redis = require('redis');
const client = redis.createClient({ url: process.env.REDIS_URL });
await client.connect();
const router = express.Router();

router.get('/jobs', async (req, res) => {
  const cacheKey = 'jobs:latest';
  const cached = await client.get(cacheKey);
  if (cached) return res.json(JSON.parse(cached));

  const jobs = await Job.find().lean();
  await client.setEx(cacheKey, 600, JSON.stringify(jobs));
  res.json(jobs);
});
```

**Commands**: `GET`, `SETEX`, `DEL`.  

**Tricky Question**: *How do you prevent cache stampede?*  
**Answer**: Use a lock with `SETNX` to ensure only one request fetches from the database:  
```javascript
const lockKey = `lock:${cacheKey}`;
if (await client.setNX(lockKey, '1')) {
  await client.expire(lockKey, 10);
  const jobs = await Job.find().lean();
  await client.setEx(cacheKey, 600, JSON.stringify(jobs));
  await client.del(lockKey);
}
```

---

### 5. How do you implement rate limiting in a Node.js application using Redis?
**Answer**:  
Rate limiting restricts API requests per user/IP within a time window using Redis’s atomic `INCR` and `EXPIRE`.  

**Steps**:  
1. Use a key like `rate:<ip>` to track requests.  
2. Increment count with `INCR`.  
3. Set TTL with `EXPIRE` for the window.  
4. Check if count exceeds the limit.  

**Example**:  
Middleware for rate limiting:  
```javascript
const redis = require('redis');
const client = redis.createClient({ url: process.env.REDIS_URL });
await client.connect();

const rateLimit = async (req, res, next) => {
  const key = `rate:${req.ip}`;
  const count = await client.incr(key);
  if (count === 1) await client.expire(key, 60); // 60s window
  if (count > 10) {
    return res.status(429).json({ message: 'Too many requests' });
  }
  next();
};

app.use(rateLimit);
```

**Commands**: `INCR`, `EXPIRE`, `TTL`.  

**Tricky Question**: *How do you handle high-concurrency rate limiting?*  
**Answer**: Use a Lua script for atomic operations:  
```lua
local key = KEYS[1]
local limit = tonumber(ARGV[1])
local window = tonumber(ARGV[2])
local count = redis.call('INCR', key)
if count == 1 then
  redis.call('EXPIRE', key, window)
end
if count > limit then
  return 0
end
return 1
```
```javascript
const result = await client.eval(script, { keys: [key], arguments: ['10', '60'] });
```

---

### 6. How do you use Redis Pub/Sub for real-time notifications in Node.js?
**Answer**:  
Redis Pub/Sub enables real-time messaging by publishing messages to channels and subscribing to them. In Node.js, use `node-redis` to publish and subscribe, often with WebSockets (e.g., `socket.io`) for client delivery.  

**Steps**:  
1. Publish messages with `PUBLISH`.  
2. Subscribe to channels with `SUBSCRIBE`.  
3. Integrate with WebSockets for client updates.  

**Example**:  
Publish notifications and subscribe in Node.js with Socket.IO:  
```javascript
const redis = require('redis');
const { Server } = require('socket.io');
const pubClient = redis.createClient({ url: process.env.REDIS_URL });
const subClient = pubClient.duplicate();
await Promise.all([pubClient.connect(), subClient.connect()]);
const io = new Server(server);

subClient.subscribe('notifications', (message) => {
  io.emit('notification', JSON.parse(message));
});

// Publish notification
app.post('/notify', async (req, res) => {
  const { message } = req.body;
  await pubClient.publish('notifications', JSON.stringify({ message }));
  res.json({ status: 'Notification sent' });
});
```

**Client-Side**:  
```javascript
const socket = io();
socket.on('notification', (data) => console.log('Notification:', data));
```

**Commands**: `PUBLISH`, `SUBSCRIBE`, `PSUBSCRIBE`.  

**Tricky Question**: *What if subscribers miss messages?*  
**Answer**: Pub/Sub doesn’t persist messages. Use Redis Streams (`XADD`, `XREAD`) for persistent messaging.

---

### 7. How do you manage sessions in Node.js with Redis?
**Answer**:  
Redis is ideal for session storage in Node.js due to its speed and TTL support. Use `connect-redis` with `express-session` to store session data.  

**Steps**:  
1. Install `express-session` and `connect-redis`: `npm install express-session connect-redis`.  
2. Configure Redis as the session store.  
3. Set session options with TTL.  

**Example**:  
```javascript
const express = require('express');
const session = require('express-session');
const RedisStore = require('connect-redis').default;
const redis = require('redis');
const client = redis.createClient({ url: process.env.REDIS_URL });
await client.connect();

const app = express();
app.use(
  session({
    store: new RedisStore({ client }),
    secret: process.env.SESSION_SECRET,
    resave: false,
    saveUninitialized: false,
    cookie: { maxAge: 24 * 60 * 60 * 1000 }, // 1 day
  })
);

app.get('/login', (req, res) => {
  req.session.userId = '123';
  res.json({ status: 'Logged in' });
});
```

**Commands**: `SETEX`, `GET`, `DEL`.  

**Tricky Question**: *How do you handle session cleanup?*  
**Answer**: Redis automatically expires sessions based on TTL. Monitor with `INFO MEMORY` and set `maxmemory-policy volatile-lru`.

---

## Advanced/Tricky Questions

### 8. How do you handle Redis transactions in Node.js?
**Answer**:  
Redis transactions ensure atomic execution using `MULTI` and `EXEC`. In Node.js, use `node-redis`’s `multi` method to queue commands.  

**Example**:  
Update a user’s profile and session atomically:  
```javascript
const multi = client.multi();
multi.hSet('user:123', { username: 'john_doe' });
multi.setEx('session:123', 3600, 'active');
const results = await multi.exec();
```

**Commands**: `MULTI`, `EXEC`, `DISCARD`, `WATCH`.  

**Tricky Question**: *What happens if a watched key changes?*  
**Answer**: If a `WATCH`ed key is modified before `EXEC`, the transaction fails (returns `null`). Retry the transaction:  
```javascript
let success = false;
while (!success) {
  await client.watch('user:123');
  const multi = client.multi();
  multi.hSet('user:123', { username: 'john_doe' });
  const results = await multi.exec();
  if (results !== null) success = true;
}
```

---

### 9. How do you optimize Redis performance in a Node.js application?
**Answer**:  
Optimize Redis in Node.js by:  
- **Pipelining**: Batch commands to reduce round-trips.  
- **Connection Management**: Reuse a single Redis client or use connection pooling.  
- **Data Structures**: Use hashes for structured data to minimize keys.  
- **TTL**: Set appropriate expiration times to manage memory.  
- **Lua Scripts**: Execute complex logic atomically.  

**Example**:  
Pipeline multiple commands:  
```javascript
const pipeline = client.pipeline();
pipeline.set('key1', 'value1');
pipeline.set('key2', 'value2');
await pipeline.exec();
```

**Lua Script Example**:  
```lua
local key = KEYS[1]
local value = redis.call('GET', key)
if value then
  redis.call('INCR', key)
  return value
end
return nil
```
```javascript
const result = await client.eval(script, { keys: ['counter'] });
```

**Commands**: `PIPELINE`, `EVAL`.  

**Tricky Question**: *How do you handle Redis memory issues?*  
**Answer**: Set `maxmemory` and `maxmemory-policy volatile-lru`, monitor with `INFO MEMORY`, and optimize data structures.

---

### 10. How do you secure Redis in a Node.js production environment?
**Answer**:  
Secure Redis by:  
- **Authentication**: Set a password with `requirepass`.  
- **TLS**: Use `rediss://` for encrypted connections.  
- **Firewall**: Restrict access to specific IPs.  
- **Disable Commands**: Rename dangerous commands (e.g., `FLUSHALL`).  
- **Environment Variables**: Store credentials securely.  

**Example**:  
Redis configuration in Node.js:  
```javascript
const redis = require('redis');
const client = redis.createClient({
  url: `rediss://:${process.env.REDIS_PASSWORD}@${process.env.REDIS_HOST}:6379`,
});
await client.connect();
```

**Redis Config**:  
```plaintext
requirepass your_secure_password
bind 127.0.0.1
rename-command FLUSHALL ""
```

**Commands**: `CONFIG SET requirepass`, `AUTH`.  

**Tricky Question**: *What if credentials are exposed?*  
**Answer**: An attacker could execute arbitrary commands. Use TLS, strong passwords, and monitor with `MONITOR`.

---

### 11. How do you implement a task queue in Node.js with Redis?
**Answer**:  
Use Redis lists as a task queue, with `LPUSH` to add tasks and `BRPOP` for workers to retrieve them. Libraries like `bull` simplify queue management.  

**Example with `node-redis`**:  
```javascript
// Producer
app.post('/tasks', async (req, res) => {
  const task = JSON.stringify(req.body);
  await client.lPush('tasks', task);
  res.json({ status: 'Task queued' });
});

// Worker
async function processTasks() {
  while (true) {
    const task = await client.brPop('tasks', 0);
    console.log('Processing:', JSON.parse(task.element));
  }
}
processTasks();
```

**Using Bull**:  
```javascript
const Queue = require('bull');
const taskQueue = new Queue('tasks', process.env.REDIS_URL);

taskQueue.process(async (job) => {
  console.log('Processing:', job.data);
});

app.post('/tasks', async (req, res) => {
  await taskQueue.add(req.body);
  res.json({ status: 'Task queued' });
});
```

**Commands**: `LPUSH`, `BRPOP`, `LLEN`.  

**Tricky Question**: *How do you handle failed tasks?*  
**Answer**: Use `bull`’s retry mechanism or move failed tasks to a `failed_tasks` list:  
```javascript
await client.lPush('failed_tasks', task);
```

---

### 12. How do you use Redis Streams in Node.js for event processing?
**Answer**:  
Redis Streams store persistent event logs, supporting consumer groups for scalable processing. In Node.js, use `XADD` to add events and `XREADGROUP` for consumers.  

**Example**:  
```javascript
// Producer
await client.xAdd('events', '*', { userId: '123', action: 'login' });

// Consumer
const group = 'event_group';
await client.xGroupCreate('events', group, '0', { MKSTREAM: true }).catch(() => {});
while (true) {
  const messages = await client.xReadGroup(group, 'consumer1', [
    { id: '>', stream: 'events' },
  ], { COUNT: 10 });
  for (const message of messages[0].messages) {
    console.log('Event:', message.fields);
    await client.xAck('events', group, message.id);
  }
}
```

**Commands**: `XADD`, `XREADGROUP`, `XGROUP`, `XACK`.  

**Tricky Question**: *How do you scale stream processing?*  
**Answer**: Add multiple consumers to the same group with `XGROUP CREATECONSUMER`.

---

## Project-Specific Questions (Node.js Job Portal Context)

### 13. How did you use Redis for caching in a Node.js job portal?
**Answer**:  
In a job portal, Redis cached job listings, user profiles, and search results to reduce MongoDB load.  

**Example**:  
Cache job search results:  
```javascript
router.get('/jobs', async (req, res) => {
  const { query, page = 1, limit = 10 } = req.query;
  const cacheKey = `jobs:search:${query}:${page}:${limit}`;
  const cached = await client.get(cacheKey);
  if (cached) return res.json(JSON.parse(cached));

  const jobs = await Job.find({ $text: { $search: query } })
    .skip((page - 1) * limit)
    .limit(Number(limit));
  await client.setEx(cacheKey, 600, JSON.stringify(jobs));
  res.json(jobs);
});
```

**Commands**: `SETEX`, `GET`, `DEL`.  

**Tricky Question**: *How do you invalidate job caches?*  
**Answer**: Clear caches on job CRUD operations:  
```javascript
await client.del('jobs:search:*');
```

---

### 14. How did you implement job notifications with Redis Pub/Sub in Node.js?
**Answer**:  
Used Redis Pub/Sub to send real-time job alerts to candidates. Published notifications when employers posted jobs, and clients received them via Socket.IO.  

**Example**:  
```javascript
router.post('/jobs', async (req, res) => {
  const job = await Job.create(req.body);
  await pubClient.publish('job_alerts', JSON.stringify({ jobId: job._id }));
  res.json(job);
});

subClient.subscribe('job_alerts', (message) => {
  io.emit('job_alert', JSON.parse(message));
});
```

**Commands**: `PUBLISH`, `SUBSCRIBE`.  

**Tricky Question**: *How do you ensure reliable delivery?*  
**Answer**: Use Redis Streams for persistent events:  
```javascript
await client.xAdd('job_alerts', '*', { jobId });
```

---

### 15. How did you use Redis for rate limiting job applications in Node.js?
**Answer**:  
Limited job applications per candidate to prevent spam using Redis `INCR` and `EXPIRE`.  

**Example**:  
```javascript
router.post('/apply/:jobId', async (req, res) => {
  const userId = req.user._id;
  const key = `apply:${userId}:${jobId}`;
  const count = await client.incr(key);
  if (count === 1) await client.expire(key, 86400); // 1 day
  if (count > 1) return res.status(429).json({ message: 'Already applied' });

  const application = await Application.create({ userId, jobId });
  res.json(application);
});
```

**Commands**: `INCR`, `EXPIRE`.  

**Tricky Question**: *How do you track application counts globally?*  
**Answer**: Use a sorted set:  
```javascript
await client.zIncrBy('applications:count', 1, userId);
```

---