# Media Storage - Deep Dive

## Step 4: Deep Dive

### 1. Chunking Strategy (Block Storage)
Why chunk large files?
- **Resumable Uploads**: If a 5GB file fails at 90%, you only retry the last chunk.
- **Deduplication**: Only store unique blocks. If two users upload the same video, we store it once.
- **Parallelism**: Upload chunks in parallel.

**Strategy:**
- Split files into fixed-size blocks (e.g., 4MB).
- Compute SHA-256 hash of each block.
- Upload block to Object Store (S3) keyed by Hash.

### 2. Deduplication (Data & Storage Efficiency)
**How it works:**
1. Client calculates hash of a 4MB block.
2. Client queries Block Service: "Do you have block `hash123`?"
3. **If Yes**: Client skips uploading. Cloud just links this block to the file's metadata.
4. **If No**: Client uploads the bytes.

**Impact:**
- Massive bandwidth savings (especially for viral content shared by many).
- Massive storage savings (dedup ratios can be 30-50% in backup systems).

### 3. Metadata & Synchronization
We need to keep the Client and Server in sync.
**Differential Sync:**
- Instead of downloading the whole file, download only changed blocks.
- **Rsync algorithm**: Rolling checksums to detect changes at byte level (too complex for MVP, stick to block level).

**Database Choice for Metadata:**
- **Requirement**: Strong consistency (ACID) for file moves/renames.
- **Choice**: **Distributed SQL (CockroachDB/TiDB)** or **Sharded MySQL/Postgres**.
- **Sharding Key**: `user_id`. All files for a user live on one shard. Allows transactional updates for "Move folder" operations.
- Avoid NoSQL (Cassandra) for metadata because "move" operations are expensive (might need to rewrite keys).

### 4. Search Service
- Separate service from Metadata DB.
- Use **Elasticsearch**.
- Async indexing (Kafka consumer reads metadata changes -> Updates ES).

## Trade-offs Discussion

| Decision | Option A | Option B | Your Choice | Why? |
|----------|----------|----------|-------------|------|
| **Storage** | File System (NFS) | Object Store (S3) | **S3** | Limitless scale, cheaper, built-in redundancy. |
| **Chunk Size** | Small (64KB) | Large (4MB) | **4MB** | Small = too much metadata. Large = worse dedup. 4MB is standard. |
| **Metadata DB** | NoSQL | SQL | **SQL** | Need ACID for folder moves and referential integrity. |

## Next Steps
Continue to [failure-scenarios.md](failure-scenarios.md)
