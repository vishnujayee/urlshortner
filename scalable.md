# URL Shortener - Enterprise Scalable Architecture

## 📋 Table of Contents
- [System Overview](#system-overview)
- [URL Shortening Algorithms](#url-shortening-algorithms)
- [Scalable Architecture Design](#scalable-architecture-design)
- [Database Design & Optimization](#database-design--optimization)
- [Caching Strategy](#caching-strategy)
- [Load Balancing & Distribution](#load-balancing--distribution)
- [Security & Rate Limiting](#security--rate-limiting)
- [Analytics & Monitoring](#analytics--monitoring)
- [Professional Features](#professional-features)
- [High Availability Setup](#high-availability-setup)
- [Performance Benchmarks](#performance-benchmarks)

## System Overview

### Architecture Principles
Our URL shortener follows enterprise-grade principles:
- **Horizontal Scalability**: Designed to scale across multiple servers
- **High Availability**: 99.9% uptime with redundancy
- **Low Latency**: Sub-100ms response times for redirects
- **Data Consistency**: ACID compliance with eventual consistency for analytics
- **Security First**: Multiple layers of protection
- **Observability**: Comprehensive monitoring and logging

### High-Level Architecture

```
┌─────────────────┬─────────────────┬─────────────────┐
│   Load Balancer │   Load Balancer │   Load Balancer │
│    (HAProxy)    │     (Nginx)     │   (CloudFlare)  │
└─────────┬───────┴─────────┬───────┴─────────┬───────┘
          │                 │                 │
┌─────────▼─────────────────▼─────────────────▼───────┐
│              API Gateway (Kong/AWS API Gateway)     │
│              • Rate Limiting                        │
│              • Authentication                       │
│              • Request Routing                      │
└─────────┬───────────────────────────────────────────┘
          │
┌─────────▼─────────────────────────────────────────────┐
│                Application Layer                      │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐            │
│  │ App Node │  │ App Node │  │ App Node │            │
│  │    1     │  │    2     │  │    N     │            │
│  └──────────┘  └──────────┘  └──────────┘            │
└─────────┬───────────────────────────────────────────┘
          │
┌─────────▼─────────────────────────────────────────────┐
│                   Caching Layer                       │
│  ┌─────────────────┐  ┌─────────────────┐            │
│  │ Redis Cluster   │  │ Memcached       │            │
│  │ (Hot URLs)      │  │ (Session Data)  │            │
│  └─────────────────┘  └─────────────────┘            │
└─────────┬───────────────────────────────────────────┘
          │
┌─────────▼─────────────────────────────────────────────┐
│                  Database Layer                       │
│  ┌─────────────────┐  ┌─────────────────┐            │
│  │ MongoDB Cluster │  │ ClickHouse      │            │
│  │ (Primary Data)  │  │ (Analytics)     │            │
│  │                 │  │                 │            │
│  │ ┌─────┐ ┌─────┐ │  │ ┌─────┐ ┌─────┐ │            │
│  │ │Shard│ │Shard│ │  │ │Node │ │Node │ │            │
│  │ │  1  │ │  2  │ │  │ │  1  │ │  2  │ │            │
│  │ └─────┘ └─────┘ │  │ └─────┘ └─────┘ │            │
│  └─────────────────┘  └─────────────────┘            │
└───────────────────────────────────────────────────────┘
```

## URL Shortening Algorithms

### 1. Current Implementation: Safe Character Set

Our production algorithm uses a carefully crafted character set:

```javascript
// Safe alphabet excluding confusing characters
const ALPHABET = 'abcdefghijkmnpqrstuvwxyzABCDEFGHJKLMNPQRSTUVWXYZ23456789';
// Excludes: 0, O, I, l, 1 (to prevent user confusion)

// Enhanced generation with collision handling
const generateShortCode = async (length = 6) => {
  const maxRetries = 5;
  let attempt = 0;
  
  while (attempt < maxRetries) {
    const code = Array.from(
      { length }, 
      () => ALPHABET.charAt(Math.floor(Math.random() * ALPHABET.length))
    ).join('');
    
    // Check for collisions and reserved words
    if (!await isCodeTaken(code) && !isReservedWord(code)) {
      return code;
    }
    
    attempt++;
    length += Math.floor(attempt / 2); // Exponential backoff
  }
  
  throw new Error('Unable to generate unique short code');
};
```

### 2. Alternative Algorithms for Scale

#### A. Base62 Encoding with Counter
```javascript
// For guaranteed uniqueness at scale
class Base62Counter {
  constructor(startValue = 1000000) {
    this.counter = startValue;
  }
  
  encode(num) {
    const base = ALPHABET.length;
    let result = '';
    
    while (num > 0) {
      result = ALPHABET[num % base] + result;
      num = Math.floor(num / base);
    }
    
    return result || ALPHABET[0];
  }
  
  getNext() {
    return this.encode(this.counter++);
  }
}
```

#### B. Hash-Based Approach
```javascript
// For distributed systems
const crypto = require('crypto');

const hashUrl = (originalUrl, userId, salt) => {
  const input = `${originalUrl}:${userId}:${salt}:${Date.now()}`;
  const hash = crypto.createHash('sha256').update(input).digest('hex');
  
  // Convert to base62 and take first 6 characters
  let result = '';
  let num = parseInt(hash.substring(0, 8), 16);
  
  while (num > 0 && result.length < 6) {
    result = ALPHABET[num % ALPHABET.length] + result;
    num = Math.floor(num / ALPHABET.length);
  }
  
  return result.padStart(6, ALPHABET[0]);
};
```

#### C. Snowflake Algorithm (Twitter-style)
```javascript
// For globally unique IDs across distributed systems
class SnowflakeGenerator {
  constructor(machineId = 1, datacenterId = 1) {
    this.machineId = machineId & 0x3FF; // 10 bits
    this.datacenterId = datacenterId & 0x1F; // 5 bits
    this.sequence = 0;
    this.lastTimestamp = -1;
    this.epoch = 1640995200000; // Custom epoch (2022-01-01)
  }
  
  generate() {
    let timestamp = Date.now();
    
    if (timestamp < this.lastTimestamp) {
      throw new Error('Clock moved backwards');
    }
    
    if (timestamp === this.lastTimestamp) {
      this.sequence = (this.sequence + 1) & 0xFFF; // 12 bits
      if (this.sequence === 0) {
        // Wait for next millisecond
        while (timestamp <= this.lastTimestamp) {
          timestamp = Date.now();
        }
      }
    } else {
      this.sequence = 0;
    }
    
    this.lastTimestamp = timestamp;
    
    // Combine: 42 bits timestamp + 5 bits datacenter + 10 bits machine + 12 bits sequence
    const id = ((timestamp - this.epoch) << 22) | 
               (this.datacenterId << 17) | 
               (this.machineId << 12) | 
               this.sequence;
    
    return this.encodeBase62(id);
  }
  
  encodeBase62(num) {
    let result = '';
    while (num > 0) {
      result = ALPHABET[num % ALPHABET.length] + result;
      num = Math.floor(num / ALPHABET.length);
    }
    return result;
  }
}
```

## Scalable Architecture Design

### 1. Microservices Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    API Gateway                          │
│                 (Authentication,                        │
│              Rate Limiting, Routing)                    │
└─────────────┬───────────────┬───────────────┬───────────┘
              │               │               │
    ┌─────────▼─────────┐ ┌───▼────────┐ ┌───▼─────────┐
    │ URL Service       │ │ Auth Service│ │Analytics    │
    │                   │ │             │ │Service      │
    │ • Shorten URLs    │ │ • JWT Tokens│ │             │
    │ • Redirect        │ │ • User Mgmt │ │ • Click     │
    │ • Validation      │ │ • Permissions│ │   Tracking  │
    │ • Custom Codes    │ │ • Rate Limits│ │ • Reports   │
    └─────────┬─────────┘ └───┬────────┘ │ • Real-time │
              │               │         │   Stats     │
    ┌─────────▼─────────┐ ┌───▼────────┐ └───┬─────────┘
    │ URL Database      │ │User Database│ ┌───▼─────────┐
    │ (MongoDB)         │ │(MongoDB)    │ │Analytics DB │
    └───────────────────┘ └─────────────┘ │(ClickHouse) │
                                          └─────────────┘
```

### 2. Database Sharding Strategy

#### Horizontal Sharding by User ID
```javascript
// Shard determination
const getShardKey = (userId) => {
  return userId % NUMBER_OF_SHARDS;
};

// MongoDB sharding configuration
const shardingConfig = {
  shardKey: { userId: 1 },
  chunks: [
    { min: { userId: 0 }, max: { userId: 1000000 }, shard: 'shard01' },
    { min: { userId: 1000000 }, max: { userId: 2000000 }, shard: 'shard02' },
    // ... more shards
  ]
};
```

#### Consistent Hashing for URL Distribution
```javascript
class ConsistentHashing {
  constructor(nodes = []) {
    this.ring = new Map();
    this.sortedHashes = [];
    this.virtualNodes = 150; // Virtual nodes per physical node
    
    nodes.forEach(node => this.addNode(node));
  }
  
  hash(key) {
    return crypto.createHash('md5').update(key).digest('hex');
  }
  
  addNode(node) {
    for (let i = 0; i < this.virtualNodes; i++) {
      const virtualKey = `${node}:${i}`;
      const hash = this.hash(virtualKey);
      this.ring.set(hash, node);
      this.sortedHashes.push(hash);
    }
    this.sortedHashes.sort();
  }
  
  getNode(key) {
    const hash = this.hash(key);
    const idx = this.sortedHashes.findIndex(h => h >= hash);
    const targetHash = idx === -1 ? this.sortedHashes[0] : this.sortedHashes[idx];
    return this.ring.get(targetHash);
  }
}
```

## Database Design & Optimization

### 1. Optimized Schema Design

```javascript
// URLs Collection - Optimized for reads
const urlSchema = {
  _id: ObjectId,
  shortCode: String,     // Indexed, unique
  originalUrl: String,   // Indexed for reverse lookup
  userId: ObjectId,      // Sharding key
  
  // Metadata
  title: String,
  description: String,
  favicon: String,
  
  // Performance optimizations
  clickCount: Number,    // Denormalized for quick access
  lastClicked: Date,     // For sorting popular URLs
  
  // Status and settings
  isActive: Boolean,
  isCustom: Boolean,
  expiresAt: Date,
  
  // Audit trail
  createdAt: Date,
  updatedAt: Date,
  
  // Analytics optimization
  dailyClicks: [{
    date: String,        // YYYY-MM-DD format
    count: Number
  }],
  
  // Geolocation summary
  topCountries: [{
    country: String,
    count: Number
  }]
};

// Clicks Collection - Separate for analytics
const clickSchema = {
  _id: ObjectId,
  shortCode: String,     // Reference to URL
  userId: ObjectId,      // For user analytics
  
  // Request metadata
  ip: String,           // Anonymized for privacy
  userAgent: String,
  referer: String,
  
  // Geolocation
  country: String,
  city: String,
  region: String,
  
  // Device detection
  device: {
    type: String,       // mobile, desktop, tablet
    os: String,
    browser: String
  },
  
  // Timestamp
  timestamp: Date,
  
  // Processing flags
  processed: Boolean,   // For batch analytics
  fraudScore: Number    // ML-based fraud detection
};
```

### 2. Advanced Indexing Strategy

```javascript
// URL Collection Indexes
db.urls.createIndex({ shortCode: 1 }, { unique: true });
db.urls.createIndex({ userId: 1, createdAt: -1 });
db.urls.createIndex({ userId: 1, clickCount: -1 });
db.urls.createIndex({ originalUrl: 1 });
db.urls.createIndex({ isActive: 1, expiresAt: 1 });
db.urls.createIndex({ lastClicked: -1 });

// Compound indexes for common queries
db.urls.createIndex({ userId: 1, isActive: 1, createdAt: -1 });
db.urls.createIndex({ shortCode: 1, isActive: 1 });

// Text search index
db.urls.createIndex({
  title: "text",
  description: "text",
  originalUrl: "text"
});

// TTL index for automatic cleanup
db.urls.createIndex({ expiresAt: 1 }, { expireAfterSeconds: 0 });

// Clicks Collection Indexes
db.clicks.createIndex({ shortCode: 1, timestamp: -1 });
db.clicks.createIndex({ userId: 1, timestamp: -1 });
db.clicks.createIndex({ timestamp: -1 });
db.clicks.createIndex({ processed: 1 });

// Geospatial index for location analytics
db.clicks.createIndex({ location: "2dsphere" });
```

## Caching Strategy

### 1. Multi-Layer Caching

```javascript
// Redis caching implementation
class CacheManager {
  constructor() {
    this.redis = new Redis.Cluster([
      { host: 'cache-node-1', port: 6379 },
      { host: 'cache-node-2', port: 6379 },
      { host: 'cache-node-3', port: 6379 }
    ]);
    
    this.localCache = new LRU({ max: 10000, ttl: 60000 }); // 1 minute local cache
  }
  
  async getUrl(shortCode) {
    // L1: Local cache (fastest)
    let url = this.localCache.get(`url:${shortCode}`);
    if (url) return url;
    
    // L2: Redis cache
    url = await this.redis.get(`url:${shortCode}`);
    if (url) {
      url = JSON.parse(url);
      this.localCache.set(`url:${shortCode}`, url);
      return url;
    }
    
    // L3: Database
    url = await URL.findOne({ shortCode, isActive: true });
    if (url) {
      // Cache for 1 hour in Redis
      await this.redis.setex(`url:${shortCode}`, 3600, JSON.stringify(url));
      this.localCache.set(`url:${shortCode}`, url);
    }
    
    return url;
  }
  
  async invalidateUrl(shortCode) {
    this.localCache.delete(`url:${shortCode}`);
    await this.redis.del(`url:${shortCode}`);
  }
}
```

### 2. Cache Warming Strategy

```javascript
// Preload hot URLs
class CacheWarmer {
  async warmPopularUrls() {
    const popularUrls = await URL.find({
      clickCount: { $gte: 100 },
      lastClicked: { $gte: new Date(Date.now() - 24 * 60 * 60 * 1000) }
    }).limit(1000);
    
    const pipeline = this.redis.pipeline();
    popularUrls.forEach(url => {
      pipeline.setex(`url:${url.shortCode}`, 3600, JSON.stringify(url));
    });
    
    await pipeline.exec();
  }
  
  startWarming() {
    // Warm cache every 10 minutes
    setInterval(() => this.warmPopularUrls(), 10 * 60 * 1000);
  }
}
```

## Load Balancing & Distribution

### 1. Application Load Balancing

```nginx
# Nginx configuration for load balancing
upstream urlshortener_backend {
    least_conn;
    server app1.example.com:5000 weight=3 max_fails=3 fail_timeout=30s;
    server app2.example.com:5000 weight=3 max_fails=3 fail_timeout=30s;
    server app3.example.com:5000 weight=2 max_fails=3 fail_timeout=30s;
    
    # Backup server
    server backup.example.com:5000 backup;
}

server {
    listen 80;
    server_name yourdomain.com;
    
    # Rate limiting
    limit_req_zone $binary_remote_addr zone=api:10m rate=100r/m;
    limit_req_zone $binary_remote_addr zone=redirect:10m rate=1000r/m;
    
    # API endpoints
    location /api/ {
        limit_req zone=api burst=20 nodelay;
        proxy_pass http://urlshortener_backend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        
        # Connection settings
        proxy_connect_timeout 5s;
        proxy_send_timeout 10s;
        proxy_read_timeout 10s;
    }
    
    # Short URL redirects (highest performance)
    location ~ ^/([a-zA-Z0-9]{4,8})$ {
        limit_req zone=redirect burst=50 nodelay;
        proxy_pass http://urlshortener_backend/$1;
        proxy_cache redirect_cache;
        proxy_cache_valid 200 1h;
        proxy_cache_valid 404 1m;
    }
}
```

### 2. Database Load Balancing

```javascript
// MongoDB connection with read replicas
const mongoOptions = {
  readPreference: 'secondaryPreferred',
  readConcern: { level: 'majority' },
  writeConcern: { w: 'majority', j: true },
  maxPoolSize: 50,
  minPoolSize: 5,
  maxIdleTimeMS: 30000,
  serverSelectionTimeoutMS: 5000,
  socketTimeoutMS: 45000,
};

// Read/Write splitting
class DatabaseManager {
  constructor() {
    this.writeDB = mongoose.createConnection(MONGO_WRITE_URI, mongoOptions);
    this.readDB = mongoose.createConnection(MONGO_READ_URI, {
      ...mongoOptions,
      readPreference: 'secondary'
    });
  }
  
  async findUrl(shortCode) {
    // Use read replica for queries
    return this.readDB.model('URL').findOne({ shortCode });
  }
  
  async createUrl(urlData) {
    // Use primary for writes
    return this.writeDB.model('URL').create(urlData);
  }
}
```

## Security & Rate Limiting

### 1. Advanced Rate Limiting

```javascript
// Sliding window rate limiter
class SlidingWindowRateLimiter {
  constructor(redisClient) {
    this.redis = redisClient;
  }
  
  async isAllowed(key, limit, windowSizeMs) {
    const now = Date.now();
    const pipeline = this.redis.pipeline();
    
    // Remove expired entries
    pipeline.zremrangebyscore(key, '-inf', now - windowSizeMs);
    
    // Count current requests
    pipeline.zcard(key);
    
    // Add current request
    pipeline.zadd(key, now, `${now}-${Math.random()}`);
    
    // Set expiry
    pipeline.expire(key, Math.ceil(windowSizeMs / 1000));
    
    const results = await pipeline.exec();
    const currentRequests = results[1][1];
    
    return currentRequests < limit;
  }
}

// Usage in middleware
const rateLimiter = new SlidingWindowRateLimiter(redis);

const createRateLimit = (limitPerWindow, windowMs) => {
  return async (req, res, next) => {
    const key = `rate_limit:${req.ip}:${req.route.path}`;
    const allowed = await rateLimiter.isAllowed(key, limitPerWindow, windowMs);
    
    if (!allowed) {
      return res.status(429).json({
        error: 'Too many requests',
        retryAfter: Math.ceil(windowMs / 1000)
      });
    }
    
    next();
  };
};
```

### 2. Security Layers

```javascript
// Advanced URL validation
class SecurityValidator {
  constructor() {
    this.maliciousDomains = new Set(); // Load from threat intelligence
    this.ipBlacklist = new Set();
  }
  
  async validateUrl(url) {
    const parsedUrl = new URL(url);
    
    // Check against malicious domains
    if (this.maliciousDomains.has(parsedUrl.hostname)) {
      throw new Error('Malicious domain detected');
    }
    
    // Prevent internal network access
    const ip = await this.resolveIP(parsedUrl.hostname);
    if (this.isInternalIP(ip)) {
      throw new Error('Internal network access not allowed');
    }
    
    // Check URL reputation
    const reputation = await this.checkUrlReputation(url);
    if (reputation.score < 0.5) {
      throw new Error('URL has poor reputation');
    }
    
    return true;
  }
  
  isInternalIP(ip) {
    const internal = [
      /^10\./,
      /^172\.(1[6-9]|2[0-9]|3[01])\./,
      /^192\.168\./,
      /^127\./,
      /^169\.254\./
    ];
    
    return internal.some(regex => regex.test(ip));
  }
}
```

## Analytics & Monitoring

### 1. Real-time Analytics Pipeline

```javascript
// ClickHouse for analytics
class AnalyticsEngine {
  constructor() {
    this.clickhouse = new ClickHouse({
      host: 'analytics-cluster.com',
      port: 8123,
      debug: false,
      basicAuth: {
        username: 'analytics_user',
        password: process.env.CLICKHOUSE_PASSWORD
      }
    });
  }
  
  async recordClick(clickData) {
    // Real-time processing
    await Promise.all([
      this.updateRedisCounters(clickData),
      this.streamToClickHouse(clickData),
      this.triggerRealTimeUpdates(clickData)
    ]);
  }
  
  async updateRedisCounters(clickData) {
    const pipeline = this.redis.pipeline();
    const today = new Date().toISOString().split('T')[0];
    
    // Update various counters
    pipeline.hincrby(`url:${clickData.shortCode}:daily`, today, 1);
    pipeline.hincrby(`user:${clickData.userId}:daily`, today, 1);
    pipeline.hincrby(`global:daily`, today, 1);
    pipeline.zincrby(`url:${clickData.shortCode}:countries`, 1, clickData.country);
    
    await pipeline.exec();
  }
  
  async getAnalytics(shortCode, timeRange) {
    const query = `
      SELECT 
        toDate(timestamp) as date,
        count() as clicks,
        uniq(ip) as unique_visitors,
        groupArray(country) as countries,
        groupArray(device_type) as devices
      FROM clicks 
      WHERE short_code = '${shortCode}'
        AND timestamp >= now() - INTERVAL ${timeRange} DAY
      GROUP BY date
      ORDER BY date
    `;
    
    return this.clickhouse.query(query).toPromise();
  }
}
```

### 2. Monitoring & Alerting

```javascript
// Comprehensive monitoring
class SystemMonitor {
  constructor() {
    this.metrics = new PrometheusRegistry();
    this.setupMetrics();
  }
  
  setupMetrics() {
    // Response time metrics
    this.responseTime = new Histogram({
      name: 'http_request_duration_seconds',
      help: 'Duration of HTTP requests in seconds',
      labelNames: ['method', 'route', 'status_code'],
      buckets: [0.01, 0.05, 0.1, 0.5, 1, 5]
    });
    
    // Business metrics
    this.urlsCreated = new Counter({
      name: 'urls_created_total',
      help: 'Total number of URLs created'
    });
    
    this.redirects = new Counter({
      name: 'redirects_total',
      help: 'Total number of redirects',
      labelNames: ['status']
    });
    
    // System metrics
    this.activeConnections = new Gauge({
      name: 'active_connections',
      help: 'Number of active connections'
    });
  }
  
  // Health check endpoint
  async healthCheck() {
    const checks = await Promise.allSettled([
      this.checkDatabase(),
      this.checkRedis(),
      this.checkExternalServices()
    ]);
    
    const health = {
      status: 'healthy',
      timestamp: new Date().toISOString(),
      services: {}
    };
    
    checks.forEach((check, index) => {
      const serviceName = ['database', 'redis', 'external'][index];
      health.services[serviceName] = {
        status: check.status === 'fulfilled' ? 'healthy' : 'unhealthy',
        responseTime: check.value?.responseTime || null,
        error: check.reason?.message || null
      };
      
      if (check.status === 'rejected') {
        health.status = 'degraded';
      }
    });
    
    return health;
  }
}
```

## Professional Features

### 1. Custom Domain Support

```javascript
// Custom domain management
class DomainManager {
  async addCustomDomain(userId, domain, shortCode) {
    // Validate domain ownership
    await this.validateDomainOwnership(domain);
    
    // Configure DNS
    await this.configureDNS(domain);
    
    // Issue SSL certificate
    const cert = await this.issueSSLCertificate(domain);
    
    // Save configuration
    await CustomDomain.create({
      userId,
      domain,
      shortCode,
      sslCertificate: cert,
      status: 'active'
    });
  }
  
  async validateDomainOwnership(domain) {
    const token = crypto.randomBytes(32).toString('hex');
    
    // Require TXT record verification
    const expectedRecord = `urlshortener-verification=${token}`;
    
    // Check DNS record
    const records = await dns.resolveTxt(domain);
    const verified = records.some(record => 
      record.includes(expectedRecord)
    );
    
    if (!verified) {
      throw new Error('Domain ownership verification failed');
    }
  }
}
```

### 2. Advanced Analytics Dashboard

```javascript
// Real-time dashboard data
class DashboardAPI {
  async getRealtimeDashboard(userId, timeRange = '24h') {
    const [
      urlStats,
      clickStats,
      geoStats,
      deviceStats,
      referrerStats
    ] = await Promise.all([
      this.getUrlStats(userId, timeRange),
      this.getClickStats(userId, timeRange),
      this.getGeoStats(userId, timeRange),
      this.getDeviceStats(userId, timeRange),
      this.getReferrerStats(userId, timeRange)
    ]);
    
    return {
      summary: {
        totalUrls: urlStats.total,
        totalClicks: clickStats.total,
        clicksToday: clickStats.today,
        topUrl: urlStats.topPerformer
      },
      charts: {
        clicksOverTime: clickStats.timeSeries,
        geoDistribution: geoStats.countries,
        deviceBreakdown: deviceStats.breakdown,
        topReferrers: referrerStats.top10
      },
      realtime: {
        activeUsers: await this.getActiveUsers(userId),
        recentClicks: await this.getRecentClicks(userId, 50)
      }
    };
  }
}
```

### 3. Enterprise Features

```javascript
// Team management
class TeamManager {
  async createTeam(ownerId, teamName) {
    const team = await Team.create({
      name: teamName,
      owner: ownerId,
      members: [{ user: ownerId, role: 'owner', permissions: ['all'] }],
      subscription: {
        plan: 'enterprise',
        limits: {
          urlsPerMonth: 1000000,
          customDomains: 10,
          teamMembers: 50
        }
      }
    });
    
    return team;
  }
  
  async inviteMember(teamId, email, role, permissions) {
    const invite = await TeamInvite.create({
      team: teamId,
      email,
      role,
      permissions,
      token: crypto.randomBytes(32).toString('hex'),
      expiresAt: new Date(Date.now() + 7 * 24 * 60 * 60 * 1000) // 7 days
    });
    
    // Send invitation email
    await this.sendInviteEmail(invite);
    
    return invite;
  }
}

// API management
class APIManager {
  async createAPIKey(userId, name, permissions, rateLimit) {
    const apiKey = await APIKey.create({
      user: userId,
      name,
      key: `us_${crypto.randomBytes(32).toString('hex')}`,
      permissions,
      rateLimit: {
        requestsPerHour: rateLimit,
        requestsPerDay: rateLimit * 24
      },
      isActive: true
    });
    
    return apiKey;
  }
  
  async validateAPIKey(key) {
    const apiKey = await APIKey.findOne({ key, isActive: true });
    
    if (!apiKey) {
      throw new Error('Invalid API key');
    }
    
    // Check rate limits
    const usage = await this.checkRateLimit(apiKey._id);
    if (usage.exceeded) {
      throw new Error('Rate limit exceeded');
    }
    
    return apiKey;
  }
}
```

## High Availability Setup

### 1. Multi-Region Deployment

```yaml
# Kubernetes deployment for high availability
apiVersion: apps/v1
kind: Deployment
metadata:
  name: urlshortener-app
spec:
  replicas: 6
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 2
      maxUnavailable: 1
  selector:
    matchLabels:
      app: urlshortener
  template:
    metadata:
      labels:
        app: urlshortener
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - urlshortener
            topologyKey: "kubernetes.io/hostname"
      containers:
      - name: app
        image: urlshortener:latest
        ports:
        - containerPort: 5000
        env:
        - name: NODE_ENV
          value: "production"
        - name: MONGODB_URI
          valueFrom:
            secretKeyRef:
              name: app-secrets
              key: mongodb-uri
        resources:
          requests:
            memory: "256Mi"
            cpu: "200m"
          limits:
            memory: "512Mi"
            cpu: "500m"
        readinessProbe:
          httpGet:
            path: /api/health
            port: 5000
          initialDelaySeconds: 5
          periodSeconds: 10
        livenessProbe:
          httpGet:
            path: /api/health
            port: 5000
          initialDelaySeconds: 15
          periodSeconds: 20
```

### 2. Disaster Recovery

```javascript
// Automated backup system
class BackupManager {
  async performBackup() {
    const timestamp = new Date().toISOString();
    
    // MongoDB backup
    const mongoBackup = await this.backupMongoDB(timestamp);
    
    // Redis backup
    const redisBackup = await this.backupRedis(timestamp);
    
    // Upload to multiple cloud providers
    await Promise.all([
      this.uploadToS3(mongoBackup, redisBackup),
      this.uploadToGCS(mongoBackup, redisBackup),
      this.uploadToAzure(mongoBackup, redisBackup)
    ]);
    
    // Verify backup integrity
    await this.verifyBackup(timestamp);
    
    // Clean old backups
    await this.cleanOldBackups();
  }
  
  async verifyBackup(timestamp) {
    // Test restore on separate environment
    const testResult = await this.testRestore(timestamp);
    
    if (!testResult.success) {
      await this.alertOps('Backup verification failed', testResult.error);
    }
  }
}
```

## Performance Benchmarks

### Target Performance Metrics

```javascript
// Performance requirements
const performanceTargets = {
  redirect: {
    p50: 50,    // 50ms
    p95: 100,   // 100ms
    p99: 200    // 200ms
  },
  apiRequests: {
    p50: 100,   // 100ms
    p95: 300,   // 300ms
    p99: 500    // 500ms
  },
  throughput: {
    redirectsPerSecond: 10000,
    apiRequestsPerSecond: 1000,
    urlCreationsPerSecond: 100
  },
  availability: {
    uptime: 99.9,           // 99.9%
    errorRate: 0.1,         // 0.1%
    mttr: 5                 // 5 minutes
  }
};

// Load testing configuration
const loadTestConfig = {
  scenarios: {
    redirect_load: {
      executor: 'constant-arrival-rate',
      rate: 10000,    // 10k redirects/second
      timeUnit: '1s',
      duration: '10m',
      preAllocatedVUs: 1000,
      maxVUs: 2000
    },
    api_load: {
      executor: 'constant-arrival-rate',
      rate: 1000,     // 1k API requests/second
      timeUnit: '1s',
      duration: '10m',
      preAllocatedVUs: 500,
      maxVUs: 1000
    }
  },
  thresholds: {
    http_req_duration: ['p(95)<200'],
    http_req_failed: ['rate<0.1'],
    http_reqs: ['rate>9000']
  }
};
```

This architecture supports:
- **10,000+ redirects/second** with sub-100ms latency
- **Horizontal scaling** across multiple regions
- **99.9% uptime** with automated failover
- **Real-time analytics** with ClickHouse
- **Enterprise features** like custom domains and team management
- **Professional security** with threat detection and rate limiting

The system is designed to compete with industry leaders like bit.ly and tinyurl while providing superior performance and features. 