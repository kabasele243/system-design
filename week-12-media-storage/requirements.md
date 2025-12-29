# Media Storage System Design - Requirements

## Problem Statement
Design a scalable media storage and file sharing service similar to Dropbox, Google Drive, or S3.

## Step 1: Clarify Requirements

### Functional Requirements
- [ ] Users can upload files (images, videos, docs).
- [ ] Users can download files.
- [ ] Users can view file metadata (name, size, type).
- [ ] Automatic synchronization across devices (optional but good to mention).
- [ ] File versioning (optional).

### Non-Functional Requirements
- [ ] **High Reliability & Durability**: Data must never be lost (99.999999999% durability).
- [ ] **High Availability**: Service should always be up.
- [ ] **Storage Scalability**: System must handle petabytes/exabytes of data.
- [ ] **Integrity**: Files must not be corrupted.

## Step 2: Estimate Scale

### Given:
- 50M Daily Active Users (DAU).
- Average 1 file upload per user per day.
- Average file size: 10MB (mix of docs and photos).
- Read:Write ratio = 1:1 (backup behavior) or 10:1 (sharing behavior). Let's assume 1:1 for pure storage.

### Calculations:

**Traffic:**
```
50M uploads/day ÷ 86400 ≈ 580 uploads/second
Peak (2x) ≈ 1,200 uploads/second
```

**Bandwidth:**
```
580 uploads/sec * 10MB = 5.8 GB/second (Ingress)
Egree (assuming 1:1) = 5.8 GB/second
Total Bandwidth: ~12 GB/s
```

**Storage (The big one):**
```
50M files * 10MB = 500TB / day
1 Year = 500TB * 365 ≈ 180 PB / year
10 Years = 1.8 Exabytes
```
*Implication: We cannot store this on a single disk or standard database. We need a distributed object store.*

## Next Steps
Continue to [high-level-design.md](high-level-design.md)
