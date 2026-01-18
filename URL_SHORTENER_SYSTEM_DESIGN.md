# URL Shortening Service System Design (like bit.ly)

## Table of Contents
1. [Overview](#overview)
2. [Functional Requirements](#functional-requirements)
3. [Non-Functional Requirements](#non-functional-requirements)
4. [Capacity Estimation](#capacity-estimation)
5. [High-Level Architecture](#high-level-architecture)
6. [API Design](#api-design)
7. [Database Design](#database-design)
8. [URL Shortening Algorithm](#url-shortening-algorithm)
9. [Caching Strategy](#caching-strategy)
10. [Rate Limiting](#rate-limiting)
11. [Analytics & Monitoring](#analytics--monitoring)
12. [Scalability & Performance](#scalability--performance)
13. [Security Considerations](#security-considerations)
14. [Deployment Architecture](#deployment-architecture)

---

## Overview

A URL shortening service converts long URLs into short, manageable links that redirect to the original URL. This design covers a production-ready system similar to bit.ly, capable of handling billions of URLs with high availability, low latency, and comprehensive analytics.

### Key Features
- Generate short URLs from long URLs
- Redirect short URLs to original URLs
- Custom short URL aliases
- Link expiration
- Analytics and click tracking
- User accounts and link management
- QR code generation
- API access

---

## Functional Requirements

### Core Features
1. **URL Shortening**: Convert long URLs to short URLs (6-8 characters)
2. **URL Redirection**: Redirect short URLs to original URLs with minimal latency (<100ms)
3. **Custom Aliases**: Allow users to create custom short URLs
4. **Link Expiration**: Support TTL (Time To Live) for URLs
5. **Analytics**: Track clicks, geographic location, referrers, devices, browsers
6. **User Management**: User accounts, authentication, link ownership
7. **Link Management**: View, edit, delete, and organize links
8. **QR Code Generation**: Generate QR codes for short URLs

### Optional Features
- Link preview before redirect
- Bulk URL shortening
- API rate limiting per user
- Branded domains
- A/B testing for links

---

## Non-Functional Requirements

### Performance
- **Read Latency**: <100ms for URL redirection (99th percentile)
- **Write Latency**: <500ms for URL creation
- **Throughput**: 10,000+ requests per second
- **Availability**: 99.99% uptime (4 nines)

### Scalability
- Support **100 million** new URLs per month
- Handle **10 billion** redirects per month
- Store **10 billion** URLs over 10 years

### Reliability
- No data loss
- Graceful degradation under high load
- Automatic failover and recovery

### Security
- Protection against malicious URLs
- DDoS protection
- Rate limiting to prevent abuse
- HTTPS enforcement

---

## Capacity Estimation

### Traffic Estimates
- **New URLs per month**: 100 million
- **New URLs per second**: 100M / (30 days × 24 hrs × 3600s) ≈ **40 URLs/sec**
- **Read:Write Ratio**: 100:1 (typical for URL shorteners)
- **Redirects per second**: 40 × 100 = **4,000 redirects/sec**

### Storage Estimates
- **URL entry size**: ~500 bytes (short URL, long URL, metadata, timestamps)
- **URLs over 10 years**: 100M/month × 12 months × 10 years = **12 billion URLs**
- **Total storage**: 12B × 500 bytes = **6 TB**

### Bandwidth Estimates
- **Write bandwidth**: 40 URLs/sec × 500 bytes = **20 KB/sec**
- **Read bandwidth**: 4,000 redirects/sec × 500 bytes = **2 MB/sec**

### Cache Memory Estimates
- **Cache 20% of hot URLs**: 12B × 0.2 × 500 bytes = **1.2 TB**
- **Daily cache**: Assuming 80-20 rule, cache top 20% = **~240 GB** (realistic for daily hot URLs)

---

## High-Level Architecture

```mermaid
graph TB
    subgraph "Client Layer"
        WebApp[Web Application]
        MobileApp[Mobile App]
        API_Client[API Clients]
    end
    
    subgraph "CDN & Load Balancing"
        CDN[CDN/Edge Cache]
        LB[Load Balancer]
    end
    
    subgraph "Application Layer"
        API_Gateway[API Gateway]
        Auth[Auth Service]
        URL_Service[URL Shortening Service]
        Redirect_Service[Redirect Service]
        Analytics_Service[Analytics Service]
        QR_Service[QR Code Service]
    end
    
    subgraph "Caching Layer"
        Redis_Cluster[Redis Cluster]
        CDN_Cache[CDN Cache]
    end
    
    subgraph "Data Layer"
        Primary_DB[(Primary PostgreSQL)]
        Read_Replicas[(Read Replicas)]
        Analytics_DB[(Analytics DB - Cassandra)]
        Object_Storage[Object Storage - S3]
    end
    
    subgraph "Message Queue"
        Kafka[Kafka/RabbitMQ]
    end
    
    subgraph "Monitoring & Logging"
        Prometheus[Prometheus]
        Grafana[Grafana]
        ELK[ELK Stack]
    end
    
    WebApp --> CDN
    MobileApp --> CDN
    API_Client --> LB
    CDN --> LB
    LB --> API_Gateway
    
    API_Gateway --> Auth
    API_Gateway --> URL_Service
    API_Gateway --> Redirect_Service
    API_Gateway --> Analytics_Service
    API_Gateway --> QR_Service
    
    URL_Service --> Redis_Cluster
    Redirect_Service --> Redis_Cluster
    URL_Service --> Primary_DB
    Redirect_Service --> Read_Replicas
    
    Analytics_Service --> Kafka
    Kafka --> Analytics_DB
    
    QR_Service --> Object_Storage
    
    URL_Service --> Kafka
    Redirect_Service --> Kafka
    
    API_Gateway --> Prometheus
    URL_Service --> ELK
    Redirect_Service --> ELK
```

### Architecture Components

#### Client Layer
- **Web Application**: React/Next.js SPA for user interface
- **Mobile App**: Native iOS/Android apps
- **API Clients**: Third-party integrations

#### CDN & Load Balancing
- **CDN**: CloudFlare/AWS CloudFront for global edge caching
- **Load Balancer**: NGINX/AWS ALB for traffic distribution

#### Application Layer
- **API Gateway**: Kong/AWS API Gateway for routing, authentication, rate limiting
- **Auth Service**: JWT-based authentication and authorization
- **URL Shortening Service**: Core service for generating short URLs
- **Redirect Service**: Handles redirection with minimal latency
- **Analytics Service**: Processes click events and generates insights
- **QR Service**: Generates QR codes for short URLs

#### Caching Layer
- **Redis Cluster**: In-memory cache for hot URLs (read-through cache)
- **CDN Cache**: Edge caching for frequently accessed redirects

#### Data Layer
- **Primary PostgreSQL**: Master database for URL mappings and user data
- **Read Replicas**: Slave databases for read scaling
- **Cassandra**: Time-series database for analytics data
- **S3**: Object storage for QR codes and static assets

#### Message Queue
- **Kafka/RabbitMQ**: Asynchronous processing for analytics events

---

## API Design

### REST API Endpoints

#### 1. Create Short URL
```http
POST /api/v1/urls
Content-Type: application/json
Authorization: Bearer <token>

Request:
{
  "long_url": "https://www.example.com/very/long/url/path",
  "custom_alias": "mylink",  // optional
  "expiration_date": "2026-12-31T23:59:59Z",  // optional
  "user_id": "user123"  // optional for anonymous
}

Response (201 Created):
{
  "short_url": "https://short.ly/abc123",
  "short_code": "abc123",
  "long_url": "https://www.example.com/very/long/url/path",
  "created_at": "2026-01-18T08:31:51Z",
  "expiration_date": "2026-12-31T23:59:59Z",
  "qr_code_url": "https://short.ly/qr/abc123.png"
}
```

#### 2. Redirect Short URL
```http
GET /{short_code}

Response (301 Moved Permanently or 302 Found):
Location: https://www.example.com/very/long/url/path
```

#### 3. Get URL Analytics
```http
GET /api/v1/urls/{short_code}/analytics
Authorization: Bearer <token>

Response (200 OK):
{
  "short_code": "abc123",
  "total_clicks": 15420,
  "clicks_by_date": [...],
  "clicks_by_country": {...},
  "clicks_by_referrer": {...},
  "clicks_by_device": {...},
  "clicks_by_browser": {...}
}
```

#### 4. Delete Short URL
```http
DELETE /api/v1/urls/{short_code}
Authorization: Bearer <token>

Response (204 No Content)
```

#### 5. Get User URLs
```http
GET /api/v1/users/{user_id}/urls?page=1&limit=20
Authorization: Bearer <token>

Response (200 OK):
{
  "urls": [...],
  "total": 150,
  "page": 1,
  "limit": 20
}
```

---

## Database Design

### PostgreSQL Schema (Primary Database)

#### URLs Table
```sql
CREATE TABLE urls (
    id BIGSERIAL PRIMARY KEY,
    short_code VARCHAR(10) UNIQUE NOT NULL,
    long_url TEXT NOT NULL,
    user_id BIGINT REFERENCES users(id),
    custom_alias BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    expiration_date TIMESTAMP,
    is_active BOOLEAN DEFAULT TRUE,
    click_count BIGINT DEFAULT 0,
    
    INDEX idx_short_code (short_code),
    INDEX idx_user_id (user_id),
    INDEX idx_created_at (created_at),
    INDEX idx_expiration_date (expiration_date)
);
```

#### Users Table
```sql
CREATE TABLE users (
    id BIGSERIAL PRIMARY KEY,
    email VARCHAR(255) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    api_key VARCHAR(64) UNIQUE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    is_premium BOOLEAN DEFAULT FALSE,
    rate_limit_tier VARCHAR(20) DEFAULT 'free',
    
    INDEX idx_email (email),
    INDEX idx_api_key (api_key)
);
```

#### Custom Domains Table
```sql
CREATE TABLE custom_domains (
    id BIGSERIAL PRIMARY KEY,
    user_id BIGINT REFERENCES users(id),
    domain VARCHAR(255) UNIQUE NOT NULL,
    verified BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    
    INDEX idx_user_id (user_id),
    INDEX idx_domain (domain)
);
```

### Cassandra Schema (Analytics Database)

#### Click Events Table
```cql
CREATE TABLE click_events (
    short_code TEXT,
    timestamp TIMESTAMP,
    ip_address TEXT,
    country TEXT,
    city TEXT,
    referrer TEXT,
    user_agent TEXT,
    device_type TEXT,
    browser TEXT,
    os TEXT,
    
    PRIMARY KEY (short_code, timestamp)
) WITH CLUSTERING ORDER BY (timestamp DESC);
```

### Database Sharding Strategy

For horizontal scaling, shard the URLs table by `short_code`:
- **Shard Key**: First 2 characters of `short_code`
- **Number of Shards**: 64 (62² for base62 encoding)
- **Consistent Hashing**: Use consistent hashing for even distribution

---

## URL Shortening Algorithm

### Approach 1: Base62 Encoding with Auto-Increment ID

```python
import string

BASE62 = string.digits + string.ascii_lowercase + string.ascii_uppercase

def encode_base62(num):
    """Convert number to base62 string"""
    if num == 0:
        return BASE62[0]
    
    result = []
    while num > 0:
        result.append(BASE62[num % 62])
        num //= 62
    
    return ''.join(reversed(result))

def generate_short_code(id):
    """Generate 6-8 character short code from database ID"""
    return encode_base62(id).zfill(6)  # Pad to 6 characters

# Example:
# ID 1 -> "000001"
# ID 62 -> "000010"
# ID 3844 -> "000100"
# ID 916132832 -> "zzzzz"  (62^6 = 56 billion combinations)
```

**Pros:**
- No collisions (guaranteed unique)
- Simple and fast
- Predictable length

**Cons:**
- Sequential IDs can be guessed
- Requires database write to get ID

### Approach 2: Hash-Based with Collision Handling

```python
import hashlib
import base64

def generate_short_code_hash(long_url, salt=""):
    """Generate short code using MD5 hash"""
    hash_input = long_url + salt
    hash_digest = hashlib.md5(hash_input.encode()).digest()
    base64_encoded = base64.urlsafe_b64encode(hash_digest).decode()
    
    # Take first 6-8 characters
    short_code = base64_encoded[:7]
    
    return short_code

# Collision handling:
# 1. Check if short_code exists in database
# 2. If exists, append counter and rehash
# 3. Retry until unique short_code found
```

**Pros:**
- No database dependency for generation
- Can generate offline
- Non-sequential

**Cons:**
- Potential collisions (requires retry logic)
- Slightly more complex

### Approach 3: Key Generation Service (KGS)

Pre-generate and store unused short codes in a separate service:

```mermaid
graph LR
    KGS[Key Generation Service] --> Unused_Keys[(Unused Keys DB)]
    KGS --> Used_Keys[(Used Keys DB)]
    URL_Service[URL Service] --> KGS
    URL_Service --> Main_DB[(Main Database)]
```

**Process:**
1. KGS pre-generates millions of random 6-8 character codes
2. Stores them in "unused_keys" table
3. When URL service needs a code, it requests from KGS
4. KGS moves code from "unused" to "used" table
5. URL service uses the code immediately

**Pros:**
- No collisions
- Very fast (no computation needed)
- Can handle high concurrency

**Cons:**
- Additional service to maintain
- Requires key management

### Recommended Approach

**Use Base62 Encoding with Distributed ID Generator (Snowflake)**

```python
# Snowflake ID structure (64 bits):
# 1 bit: unused
# 41 bits: timestamp (milliseconds since epoch)
# 10 bits: machine ID
# 12 bits: sequence number

class SnowflakeIDGenerator:
    def __init__(self, machine_id):
        self.machine_id = machine_id
        self.sequence = 0
        self.last_timestamp = -1
        self.epoch = 1609459200000  # 2021-01-01 00:00:00
    
    def generate_id(self):
        timestamp = self._current_timestamp()
        
        if timestamp < self.last_timestamp:
            raise Exception("Clock moved backwards")
        
        if timestamp == self.last_timestamp:
            self.sequence = (self.sequence + 1) & 4095
            if self.sequence == 0:
                timestamp = self._wait_next_millis(timestamp)
        else:
            self.sequence = 0
        
        self.last_timestamp = timestamp
        
        id = ((timestamp - self.epoch) << 22) | \
             (self.machine_id << 12) | \
             self.sequence
        
        return id
    
    def _current_timestamp(self):
        import time
        return int(time.time() * 1000)
    
    def _wait_next_millis(self, last_timestamp):
        timestamp = self._current_timestamp()
        while timestamp <= last_timestamp:
            timestamp = self._current_timestamp()
        return timestamp

# Usage:
id_generator = SnowflakeIDGenerator(machine_id=1)
unique_id = id_generator.generate_id()
short_code = encode_base62(unique_id)
```

**Benefits:**
- Distributed ID generation (no single point of failure)
- Time-ordered IDs
- No database dependency
- 4096 IDs per millisecond per machine
- Supports 1024 machines

---

## Caching Strategy

### Multi-Layer Caching

```mermaid
graph TD
    Request[Redirect Request] --> CDN_Check{CDN Cache Hit?}
    CDN_Check -->|Yes| CDN_Return[Return from CDN]
    CDN_Check -->|No| Redis_Check{Redis Cache Hit?}
    Redis_Check -->|Yes| Redis_Return[Return from Redis + Update CDN]
    Redis_Check -->|No| DB_Query[Query Database]
    DB_Query --> Update_Redis[Update Redis Cache]
    Update_Redis --> Update_CDN[Update CDN Cache]
    Update_CDN --> Return_Response[Return Response]
```

### Cache Configuration

#### CDN Cache (CloudFlare/CloudFront)
```nginx
# Cache-Control headers for redirect responses
location ~ ^/[a-zA-Z0-9]{6,8}$ {
    add_header Cache-Control "public, max-age=3600, s-maxage=86400";
    add_header X-Cache-Status $upstream_cache_status;
}
```

- **TTL**: 24 hours for edge cache
- **Cache Key**: Short code
- **Invalidation**: On URL deletion or expiration

#### Redis Cache
```python
import redis
import json

redis_client = redis.Redis(host='localhost', port=6379, db=0)

def get_url_from_cache(short_code):
    """Get URL from Redis cache"""
    cached = redis_client.get(f"url:{short_code}")
    if cached:
        return json.loads(cached)
    return None

def set_url_in_cache(short_code, url_data, ttl=3600):
    """Set URL in Redis cache with TTL"""
    redis_client.setex(
        f"url:{short_code}",
        ttl,
        json.dumps(url_data)
    )

def invalidate_cache(short_code):
    """Invalidate cache entry"""
    redis_client.delete(f"url:{short_code}")
```

- **TTL**: 1 hour for hot URLs
- **Eviction Policy**: LRU (Least Recently Used)
- **Cache Size**: 240 GB (20% of daily traffic)

### Cache Warming

Pre-populate cache with trending URLs:

```python
def warm_cache():
    """Warm cache with top URLs from last 24 hours"""
    top_urls = get_top_urls_last_24h(limit=10000)
    for url in top_urls:
        set_url_in_cache(url['short_code'], url)
```

---

## Rate Limiting

### Multi-Tier Rate Limiting Strategy

```python
from datetime import datetime, timedelta
import redis

class RateLimiter:
    def __init__(self, redis_client):
        self.redis = redis_client
        self.tiers = {
            'free': {'requests_per_hour': 100, 'requests_per_day': 1000},
            'basic': {'requests_per_hour': 1000, 'requests_per_day': 10000},
            'premium': {'requests_per_hour': 10000, 'requests_per_day': 100000},
            'enterprise': {'requests_per_hour': 100000, 'requests_per_day': 1000000}
        }
    
    def is_allowed(self, user_id, tier='free'):
        """Check if user is within rate limits"""
        hour_key = f"ratelimit:{user_id}:hour:{datetime.now().hour}"
        day_key = f"ratelimit:{user_id}:day:{datetime.now().date()}"
        
        hour_count = self.redis.incr(hour_key)
        day_count = self.redis.incr(day_key)
        
        # Set expiration on first request
        if hour_count == 1:
            self.redis.expire(hour_key, 3600)
        if day_count == 1:
            self.redis.expire(day_key, 86400)
        
        limits = self.tiers[tier]
        
        if hour_count > limits['requests_per_hour']:
            return False, 'Hour limit exceeded'
        if day_count > limits['requests_per_day']:
            return False, 'Day limit exceeded'
        
        return True, 'OK'
```

### API Gateway Rate Limiting

```yaml
# Kong API Gateway configuration
plugins:
  - name: rate-limiting
    config:
      minute: 60
      hour: 1000
      policy: redis
      redis_host: redis-cluster
      fault_tolerant: true
      hide_client_headers: false
```

### DDoS Protection

- **CloudFlare**: WAF rules, bot detection, challenge pages
- **AWS Shield**: DDoS protection at network layer
- **IP Blacklisting**: Automatic blocking of malicious IPs

---

## Analytics & Monitoring

### Click Event Processing Pipeline

```mermaid
graph LR
    Redirect[Redirect Request] --> Async_Event[Async Event Publish]
    Async_Event --> Kafka[Kafka Topic]
    Kafka --> Stream_Processor[Stream Processor - Kafka Streams]
    Stream_Processor --> Cassandra[(Cassandra)]
    Stream_Processor --> Redis_Analytics[Redis Analytics Cache]
    Cassandra --> Analytics_API[Analytics API]
    Redis_Analytics --> Analytics_API
```

### Analytics Data Model

```python
# Click event structure
click_event = {
    'short_code': 'abc123',
    'timestamp': '2026-01-18T08:31:51Z',
    'ip_address': '192.168.1.1',
    'country': 'US',
    'city': 'San Francisco',
    'referrer': 'https://twitter.com',
    'user_agent': 'Mozilla/5.0...',
    'device_type': 'mobile',
    'browser': 'Chrome',
    'os': 'iOS'
}
```

### Real-Time Analytics with Kafka Streams

```java
// Kafka Streams topology for real-time aggregation
StreamsBuilder builder = new StreamsBuilder();

KStream<String, ClickEvent> clicks = builder.stream("click-events");

// Aggregate clicks by short_code and time window
KTable<Windowed<String>, Long> clickCounts = clicks
    .groupByKey()
    .windowedBy(TimeWindows.of(Duration.ofMinutes(5)))
    .count();

// Store aggregated data
clickCounts.toStream()
    .to("click-counts-aggregated");
```

### Monitoring Stack

#### Prometheus Metrics
```yaml
# Key metrics to track
- url_creation_rate
- url_redirect_rate
- cache_hit_ratio
- api_latency_p50
- api_latency_p95
- api_latency_p99
- error_rate_4xx
- error_rate_5xx
- database_connection_pool_usage
- redis_memory_usage
```

#### Grafana Dashboards
- **System Health**: CPU, memory, disk, network
- **Application Metrics**: Request rates, latencies, error rates
- **Business Metrics**: URLs created, redirects, top URLs
- **Cache Performance**: Hit/miss ratios, eviction rates

#### Alerting Rules
```yaml
# Prometheus alerting rules
groups:
  - name: url_shortener_alerts
    rules:
      - alert: HighErrorRate
        expr: rate(http_requests_total{status=~"5.."}[5m]) > 0.05
        for: 5m
        annotations:
          summary: "High error rate detected"
      
      - alert: HighLatency
        expr: histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[5m])) > 0.5
        for: 5m
        annotations:
          summary: "High latency detected (p99 > 500ms)"
      
      - alert: LowCacheHitRatio
        expr: redis_cache_hit_ratio < 0.8
        for: 10m
        annotations:
          summary: "Cache hit ratio below 80%"
```

---

## Scalability & Performance

### Horizontal Scaling Strategy

```mermaid
graph TB
    subgraph "Region 1 - US East"
        LB1[Load Balancer]
        App1[App Server 1]
        App2[App Server 2]
        App3[App Server 3]
        DB1[(Primary DB)]
        Redis1[Redis Cluster]
    end
    
    subgraph "Region 2 - EU West"
        LB2[Load Balancer]
        App4[App Server 4]
        App5[App Server 5]
        DB2[(Read Replica)]
        Redis2[Redis Cluster]
    end
    
    subgraph "Region 3 - Asia Pacific"
        LB3[Load Balancer]
        App6[App Server 6]
        App7[App Server 7]
        DB3[(Read Replica)]
        Redis3[Redis Cluster]
    end
    
    CDN[Global CDN]
    
    CDN --> LB1
    CDN --> LB2
    CDN --> LB3
    
    LB1 --> App1
    LB1 --> App2
    LB1 --> App3
    
    LB2 --> App4
    LB2 --> App5
    
    LB3 --> App6
    LB3 --> App7
    
    App1 --> DB1
    App2 --> DB1
    App3 --> DB1
    
    App4 --> DB2
    App5 --> DB2
    
    App6 --> DB3
    App7 --> DB3
    
    DB1 -.Replication.-> DB2
    DB1 -.Replication.-> DB3
```

### Database Scaling

#### Read Scaling
- **Read Replicas**: 3-5 read replicas per region
- **Connection Pooling**: PgBouncer for connection management
- **Query Optimization**: Indexed queries, query caching

#### Write Scaling
- **Database Sharding**: Shard by `short_code` prefix
- **Write-Ahead Logging**: PostgreSQL WAL for durability
- **Async Replication**: Asynchronous replication to replicas

### Application Scaling

#### Auto-Scaling Configuration
```yaml
# Kubernetes HPA (Horizontal Pod Autoscaler)
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: url-service-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: url-service
  minReplicas: 3
  maxReplicas: 50
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 80
```

### Performance Optimization

#### Database Optimization
```sql
-- Partitioning URLs table by creation date
CREATE TABLE urls_2026_01 PARTITION OF urls
    FOR VALUES FROM ('2026-01-01') TO ('2026-02-01');

-- Materialized view for analytics
CREATE MATERIALIZED VIEW top_urls AS
SELECT short_code, long_url, click_count
FROM urls
WHERE created_at > NOW() - INTERVAL '30 days'
ORDER BY click_count DESC
LIMIT 10000;

-- Refresh materialized view periodically
REFRESH MATERIALIZED VIEW CONCURRENTLY top_urls;
```

#### Connection Pooling
```python
# SQLAlchemy connection pool configuration
from sqlalchemy import create_engine
from sqlalchemy.pool import QueuePool

engine = create_engine(
    'postgresql://user:pass@localhost/urlshortener',
    poolclass=QueuePool,
    pool_size=20,
    max_overflow=40,
    pool_timeout=30,
    pool_recycle=3600
)
```

---

## Security Considerations

### URL Validation & Sanitization

```python
import validators
import re
from urllib.parse import urlparse

class URLValidator:
    BLACKLISTED_DOMAINS = [
        'malware.com',
        'phishing-site.com',
        # Load from database or external service
    ]
    
    BLACKLISTED_PATTERNS = [
        r'.*\.exe$',
        r'.*\.bat$',
        r'.*malware.*',
    ]
    
    def validate_url(self, url):
        """Validate and sanitize URL"""
        # Check if valid URL format
        if not validators.url(url):
            raise ValueError("Invalid URL format")
        
        # Parse URL
        parsed = urlparse(url)
        
        # Check protocol (only allow http/https)
        if parsed.scheme not in ['http', 'https']:
            raise ValueError("Only HTTP/HTTPS protocols allowed")
        
        # Check against blacklisted domains
        if parsed.netloc in self.BLACKLISTED_DOMAINS:
            raise ValueError("Domain is blacklisted")
        
        # Check against malicious patterns
        for pattern in self.BLACKLISTED_PATTERNS:
            if re.match(pattern, url, re.IGNORECASE):
                raise ValueError("URL matches malicious pattern")
        
        # Check with external threat intelligence API
        if self.check_threat_intelligence(url):
            raise ValueError("URL flagged by threat intelligence")
        
        return True
    
    def check_threat_intelligence(self, url):
        """Check URL against threat intelligence services"""
        # Integration with Google Safe Browsing API, VirusTotal, etc.
        # Return True if URL is malicious
        pass
```

### Authentication & Authorization

```python
import jwt
from datetime import datetime, timedelta
from passlib.hash import bcrypt

class AuthService:
    SECRET_KEY = "your-secret-key"
    
    def create_user(self, email, password):
        """Create new user with hashed password"""
        password_hash = bcrypt.hash(password)
        # Store in database
        return user_id
    
    def authenticate(self, email, password):
        """Authenticate user and return JWT token"""
        user = self.get_user_by_email(email)
        
        if not user or not bcrypt.verify(password, user['password_hash']):
            raise ValueError("Invalid credentials")
        
        # Generate JWT token
        payload = {
            'user_id': user['id'],
            'email': user['email'],
            'tier': user['rate_limit_tier'],
            'exp': datetime.utcnow() + timedelta(hours=24)
        }
        
        token = jwt.encode(payload, self.SECRET_KEY, algorithm='HS256')
        return token
    
    def verify_token(self, token):
        """Verify JWT token and return payload"""
        try:
            payload = jwt.decode(token, self.SECRET_KEY, algorithms=['HS256'])
            return payload
        except jwt.ExpiredSignatureError:
            raise ValueError("Token expired")
        except jwt.InvalidTokenError:
            raise ValueError("Invalid token")
```

### HTTPS Enforcement

```nginx
# NGINX configuration for HTTPS redirect
server {
    listen 80;
    server_name short.ly;
    return 301 https://$server_name$request_uri;
}

server {
    listen 443 ssl http2;
    server_name short.ly;
    
    ssl_certificate /etc/ssl/certs/short.ly.crt;
    ssl_certificate_key /etc/ssl/private/short.ly.key;
    
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;
    ssl_prefer_server_ciphers on;
    
    # HSTS header
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
    
    location / {
        proxy_pass http://app_servers;
    }
}
```

### SQL Injection Prevention

```python
# Use parameterized queries with SQLAlchemy
from sqlalchemy import text

def get_url_by_short_code(short_code):
    """Safe query using parameterized statement"""
    query = text("SELECT * FROM urls WHERE short_code = :code")
    result = db.execute(query, {'code': short_code})
    return result.fetchone()

# NEVER do this:
# query = f"SELECT * FROM urls WHERE short_code = '{short_code}'"  # VULNERABLE!
```

### CSRF Protection

```python
from flask_wtf.csrf import CSRFProtect

app = Flask(__name__)
app.config['SECRET_KEY'] = 'your-secret-key'
csrf = CSRFProtect(app)

# CSRF token will be automatically validated for POST/PUT/DELETE requests
```

---

## Deployment Architecture

### Docker Compose (Development)

```yaml
version: '3.8'

services:
  # Application Services
  url-service:
    build: ./services/url-service
    ports:
      - "3000:3000"
    environment:
      - DATABASE_URL=postgresql://postgres:password@postgres:5432/urlshortener
      - REDIS_URL=redis://redis:6379
      - KAFKA_BROKERS=kafka:9092
    depends_on:
      - postgres
      - redis
      - kafka
  
  redirect-service:
    build: ./services/redirect-service
    ports:
      - "3001:3001"
    environment:
      - DATABASE_URL=postgresql://postgres:password@postgres:5432/urlshortener
      - REDIS_URL=redis://redis:6379
    depends_on:
      - postgres
      - redis
  
  analytics-service:
    build: ./services/analytics-service
    ports:
      - "3002:3002"
    environment:
      - CASSANDRA_HOSTS=cassandra
      - KAFKA_BROKERS=kafka:9092
    depends_on:
      - cassandra
      - kafka
  
  # Databases
  postgres:
    image: postgres:15
    environment:
      - POSTGRES_DB=urlshortener
      - POSTGRES_USER=postgres
      - POSTGRES_PASSWORD=password
    volumes:
      - postgres_data:/var/lib/postgresql/data
    ports:
      - "5432:5432"
  
  cassandra:
    image: cassandra:4.1
    volumes:
      - cassandra_data:/var/lib/cassandra
    ports:
      - "9042:9042"
  
  # Cache
  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data
  
  # Message Queue
  zookeeper:
    image: confluentinc/cp-zookeeper:7.5.0
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181
  
  kafka:
    image: confluentinc/cp-kafka:7.5.0
    depends_on:
      - zookeeper
    ports:
      - "9092:9092"
    environment:
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka:9092
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
  
  # API Gateway
  kong:
    image: kong:3.4
    environment:
      - KONG_DATABASE=off
      - KONG_DECLARATIVE_CONFIG=/kong/kong.yml
      - KONG_PROXY_ACCESS_LOG=/dev/stdout
      - KONG_ADMIN_ACCESS_LOG=/dev/stdout
      - KONG_PROXY_ERROR_LOG=/dev/stderr
      - KONG_ADMIN_ERROR_LOG=/dev/stderr
    ports:
      - "8000:8000"
      - "8443:8443"
      - "8001:8001"
    volumes:
      - ./kong.yml:/kong/kong.yml

volumes:
  postgres_data:
  cassandra_data:
  redis_data:
```

### Kubernetes Deployment (Production)

```yaml
# url-service-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: url-service
  namespace: url-shortener
spec:
  replicas: 5
  selector:
    matchLabels:
      app: url-service
  template:
    metadata:
      labels:
        app: url-service
    spec:
      containers:
        - name: url-service
          image: url-shortener/url-service:v1.0.0
          ports:
            - containerPort: 3000
          env:
            - name: DATABASE_URL
              valueFrom:
                secretKeyRef:
                  name: db-credentials
                  key: url
            - name: REDIS_URL
              value: "redis://redis-cluster:6379"
          resources:
            requests:
              memory: "256Mi"
              cpu: "250m"
            limits:
              memory: "512Mi"
              cpu: "500m"
          livenessProbe:
            httpGet:
              path: /health
              port: 3000
            initialDelaySeconds: 30
            periodSeconds: 10
          readinessProbe:
            httpGet:
              path: /ready
              port: 3000
            initialDelaySeconds: 5
            periodSeconds: 5

---
apiVersion: v1
kind: Service
metadata:
  name: url-service
  namespace: url-shortener
spec:
  selector:
    app: url-service
  ports:
    - protocol: TCP
      port: 80
      targetPort: 3000
  type: ClusterIP
```

### Infrastructure as Code (Terraform)

```hcl
# AWS infrastructure
provider "aws" {
  region = "us-east-1"
}

# VPC
resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"
  
  tags = {
    Name = "url-shortener-vpc"
  }
}

# RDS PostgreSQL
resource "aws_db_instance" "postgres" {
  identifier           = "url-shortener-db"
  engine              = "postgres"
  engine_version      = "15.3"
  instance_class      = "db.r6g.xlarge"
  allocated_storage   = 100
  storage_type        = "gp3"
  
  db_name  = "urlshortener"
  username = "admin"
  password = var.db_password
  
  multi_az               = true
  backup_retention_period = 7
  
  tags = {
    Name = "url-shortener-postgres"
  }
}

# ElastiCache Redis
resource "aws_elasticache_cluster" "redis" {
  cluster_id           = "url-shortener-redis"
  engine               = "redis"
  node_type            = "cache.r6g.large"
  num_cache_nodes      = 3
  parameter_group_name = "default.redis7"
  port                 = 6379
  
  tags = {
    Name = "url-shortener-redis"
  }
}

# EKS Cluster
resource "aws_eks_cluster" "main" {
  name     = "url-shortener-cluster"
  role_arn = aws_iam_role.eks_cluster.arn
  
  vpc_config {
    subnet_ids = aws_subnet.private[*].id
  }
}

# Application Load Balancer
resource "aws_lb" "main" {
  name               = "url-shortener-alb"
  internal           = false
  load_balancer_type = "application"
  security_groups    = [aws_security_group.alb.id]
  subnets            = aws_subnet.public[*].id
  
  tags = {
    Name = "url-shortener-alb"
  }
}
```

---

## Summary

This system design provides a comprehensive architecture for a URL shortening service similar to bit.ly, capable of:

- **Handling billions of URLs** with efficient storage and retrieval
- **Sub-100ms redirect latency** through multi-layer caching
- **High availability** with 99.99% uptime through redundancy and failover
- **Horizontal scalability** across multiple regions
- **Comprehensive analytics** with real-time click tracking
- **Enterprise-grade security** with rate limiting, authentication, and threat protection
- **Production-ready deployment** with Docker, Kubernetes, and IaaC

### Key Design Decisions

1. **Base62 Encoding with Snowflake IDs**: Ensures unique, distributed ID generation
2. **Multi-Layer Caching**: CDN + Redis for optimal read performance
3. **Microservices Architecture**: Separate services for URL creation, redirection, and analytics
4. **PostgreSQL + Cassandra**: SQL for transactional data, NoSQL for time-series analytics
5. **Kafka for Event Streaming**: Asynchronous analytics processing
6. **Global CDN**: Edge caching for minimal latency worldwide

### Trade-offs

| Aspect | Choice | Trade-off |
|--------|--------|-----------|
| ID Generation | Snowflake | More complex but distributed and scalable |
| Database | PostgreSQL | ACID compliance but requires sharding for scale |
| Caching | Multi-layer | Higher complexity but better performance |
| Analytics | Cassandra | Eventually consistent but highly scalable |
| Architecture | Microservices | Operational overhead but better scalability |

### Future Enhancements

- **Machine Learning**: Predict trending URLs for proactive caching
- **Blockchain Integration**: Immutable URL records for compliance
- **GraphQL API**: More flexible API for complex queries
- **Serverless Functions**: Edge computing for ultra-low latency
- **Advanced Analytics**: A/B testing, conversion tracking, attribution modeling

