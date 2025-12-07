# Dynamic Replication in Haystack-Based Blob Storage System

A distributed, production-grade photo/blob storage system inspired by Facebook's Haystack architecture.  
This system implements leader election, dynamic replication, compaction, health monitoring, caching, and multiple high-availability improvements.

---

## System Architecture

```
                ┌─────────────┐
                │   Client    │
                └──────┬──────┘
                       │
                ┌──────▼──────┐
                │  Nginx LB   │
                └──────┬──────┘
                       │
     ┌─────────────────┼─────────────────┐
     │                 │                 │
┌────▼────┐       ┌────▼────┐      ┌────▼────┐
│Directory│       │Directory│      │Directory│
│   -1    │       │   -2    │      │   -3    │
└────┬────┘       └────┬────┘      └────┬────┘
     │                 │                 │
     └─────────────────┼─────────────────┘
                       │
     ┌─────────────────┼─────────────────┐
     │                 │                 │
┌────▼────┐       ┌────▼────┐      ┌────▼────┐
│ Store-1 │       │ Store-2 │      │ Store-3 │
└─────────┘       └─────────┘      └─────────┘
     │                 │                 │
     └─────────────────┼─────────────────┘
                       │
                  ┌────▼────┐
                  │  Redis  │
                  └─────────┘
                       │
                  ┌────▼─────────┐
                  │ Replication  │
                  │   Manager    │
                  └──────────────┘
```

---

## Architecture Diagrams

### Soft Delete Flow
![Soft Delete](images/soft-delete.png)

### Compaction Process
![Compaction](images/compaction.png)

### Dynamic Replication
![Replication](images/replication.png)

### Leader Election
![Leader Election](images/leader-election.png)

### Read Path
![Read Path](images/read-path.png)

### Health Monitoring
![Directory Monitor](images/directory-monitor.png)

### Upload Flow
![Upload Flow](images/upload-flow.png)

---

## Implemented Improvements

### Critical Improvements (Data Integrity & Availability)

#### 1. Tombstone Deletes
Deletes now write tombstones (`flags=1`, `size=0`) to ensure durability across restarts.

```python
tombstone = Needle(photo_id, b'', deleted=True)
with open(self.volume_path, 'ab') as f:
    f.write(tombstone.serialize())
    f.flush()
    os.fsync(f.fileno())
```

#### 2. Redis-Based Leader Election
Leader election moved to Redis using SET NX with TTL to avoid single points of failure.

```python
result = redis_client.set(self.lock_key, claim, nx=True, ex=self.ttl)
```

#### 3. Health-Aware Location Filtering
Directory now returns only healthy stores based on heartbeat freshness.

```python
if store.status == "HEALTHY" and time.time() - store.last_heartbeat < 60:
    healthy_locations.append(loc)
```

#### 4. Nightly Full Audit
Full audit worker scans all metadata to detect under-replication.

#### 5. Garbage Collection Worker
GC worker removes orphaned data not linked to directory metadata.

### High Priority Improvements (Performance & Consistency)

#### 6. Redis Cache (5GB LRU)
Replaced in-memory Python cache with Redis.

#### 7. Nginx Load Balancer
Load balances requests across directory instances.

#### 8. Push-Based Metadata Sync
Leader pushes updates, improving read-your-writes consistency.

### Medium Priority Improvements (Efficiency & Scalability)

#### 9. Push-on-Write Caching
Store pushes uploaded objects into Redis cache.

#### 10. Dynamic Store Discovery
Replication manager fetches store list from /stores endpoint.

#### 11. Reduced Stats Window
Shortened from 300s to 60s for more responsive replication decisions.

#### 12. De-Replication
Automatically removes excess replicas to reclaim storage.

### Bonus Improvements

#### 13. Rate Limiting
SlowAPI limits uploads to avoid abuse.

#### 14. Checksums
SHA256 checksums stored in metadata for integrity checks.

---

## Quick Start

### Prerequisites
- Docker and Docker Compose
- 20GB free disk space
- 8GB RAM

### Setup

```bash
git clone https://github.com/Rajender76/Haystack-Dynamic-Replication.git
cd Haystack-Dynamic-Replication
docker-compose up -d
```

Check initialization:

```bash
docker-compose logs -f
docker exec haystack-client python check_status.py
```

---

## Usage Examples

### Upload Photo
```bash
docker exec haystack-client python client_service.py upload /images/photo.jpg
```

### Download
```bash
docker exec haystack-client python client_service.py download <photo_id> /tmp/output.jpg
```

### Status
```bash
docker exec haystack-client python client_service.py status <photo_id>
```

### System Stats
```bash
docker exec haystack-client python client_service.py stats
```

---

## Configuration

Environment variables include:

**Store Service**
```
MAX_VOLUME_SIZE
COMPACTION_EFFICIENCY_THRESHOLD
HEARTBEAT_INTERVAL
```

**Directory Service**
```
LEADER_TIMEOUT
FOLLOWER_SYNC_INTERVAL
```

**Replication Manager**
```
DEFAULT_REPLICA_COUNT
MAX_REPLICA_COUNT
NIGHTLY_AUDIT_HOUR
```

**Cache**
```
CACHE_TTL
```

---

## Performance Benchmarks

- Writes: 100–200 ops/sec
- Reads (cache hit): 1000+ ops/sec
- Reads (cache miss): 200–300 ops/sec
- Failover time: <10 seconds
- Cache hit rate after warmup: 70–90%

---

## Security Considerations

- Rate limiting for write endpoints
- Checksums for data integrity
- TTL-based leader election
- Health monitoring for failover

---

## Learning Outcomes

Demonstrates engineering concepts including:

- Leader election
- Metadata consistency
- Compaction and GC
- Dynamic replication
- Load balancing
- Distributed caching
- Fault detection
- Scalable storage architecture

---

## Future Enhancements

- Multi-region replication
- Erasure coding
- Metrics export (Prometheus)
- Admin dashboard
- Encryption at rest
- Distributed tracing

---

## Contributing

Contributions are welcome.  
Please open issues or submit pull requests.

---

## Done By Team

- Adhivishnu Modapu
- Sriram Metla
- Annam Rajender Reddy

