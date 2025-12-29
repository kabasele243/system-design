# Media Storage - High-Level Design

## Step 3: High-Level Architecture

### Components

```
[Client] 
   │
   ├─(Metadata)─→ [Load Balancer] → [API Gateway] → [Metadata Service] → [Metadata DB]
   │
   └─(File Data)─→ [Load Balancer] → [Block/Upload Service] 
                                          │
                                          ↓
                                    [Obj Storage (S3)]
```

### Core Components:

1.  **Client Application**:
    *   Chunks large files.
    *   Compresses and encrypts data.
    *   Talks to Metadata Service for file info.
    *   Talks to Block Service for actual data.

2.  **Metadata Service**:
    *   Stateless API.
    *   Stores file structure, permissions, and version history.
    *   Does NOT store actual binary data.

3.  **Metadata Database**:
    *   Relational (PostgreSQL) or NoSQL (DynamoDB).
    *   Stores: `FileID`, `UserID`, `Path`, `Size`, `List<BlockID>`.

4.  **Block/Upload Service**:
    *   Receives file chunks (blocks).
    *   Computes checksums (MD5/SHA).
    *   Stores blocks in Object Storage.
    *   Deduplication logic happens here.

5.  **Object Storage (S3/GCS/Azure Blob)**:
    *   Stores the actual immutable data blocks.
    *   Handles replication and durability at the hardware level.

## API Design

### Upload File
```
1. POST /api/v1/files/upload_init
   Body: { name: "video.mp4", size: 50MB }
   Response: { upload_id: "xyz", chunk_size: 4MB }

2. PUT /api/v1/files/upload_part
   Query: { upload_id: "xyz", part_num: 1 }
   Body: [Binary Data]

3. POST /api/v1/files/upload_complete
   Body: { upload_id: "xyz", checksums: ["hash1", "hash2"] }
```

### Download File
```
GET /api/v1/files/{file_id}
Response: Streams file content
(Or returns a pre-signed URL to download directly from S3)
```

## Database Schema (Metadata)

### Files Table
```sql
CREATE TABLE files (
  file_id BIGINT PRIMARY KEY,
  user_id BIGINT,
  name VARCHAR(255),
  is_directory BOOLEAN,
  parent_id BIGINT,
  created_at TIMESTAMP
);
```

### File_Versions Table
```sql
CREATE TABLE file_versions (
  version_id BIGINT PRIMARY KEY,
  file_id BIGINT,
  version_num INT,
  s3_key VARCHAR(255), -- Or list of block IDs
  size BIGINT,
  checksum VARCHAR(64)
);
```

## Next Steps
Continue to [deep-dive.md](deep-dive.md)
