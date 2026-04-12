# Blob & Object Storage

## Object Storage

Stores data as objects (blobs) with metadata, accessed via HTTP APIs. Designed for massive scale, high durability, and cost-effective storage of unstructured data.

### Key Properties

| Property | Detail |
|----------|--------|
| **Durability** | 99.999999999% (11 nines) for S3 — designed to never lose data |
| **Availability** | 99.99% for S3 Standard |
| **Scalability** | Virtually unlimited storage |
| **Access pattern** | Read-heavy; write-once, read-many |
| **Interface** | HTTP REST API (PUT, GET, DELETE) |
| **Consistency** | Strong read-after-write (S3 since Dec 2020) |

### Object Storage vs. File Storage vs. Block Storage

| Type | Unit | Access | Use Case |
|------|------|--------|----------|
| **Object** | Objects with metadata | HTTP API | Images, videos, backups, data lakes |
| **File** | Files in directories | NFS/SMB | Shared file systems, CMS |
| **Block** | Fixed-size blocks | Attached to compute | Databases, OS volumes |

## Amazon S3 Architecture

### Organization

```
Bucket → Prefix (virtual directory) → Object (key-value)
```

- **Bucket:** Globally unique namespace, configured per-region
- **Key:** The full path (e.g., `images/2024/photo.jpg`)
- **Value:** The data (up to 5 TB per object)
- **Metadata:** System metadata (size, content-type) + custom key-value pairs

### Storage Tiers

| Tier | Access Frequency | Latency | Cost (relative) |
|------|-----------------|---------|-----------------|
| **Standard** | Frequent | ms | $$$ |
| **Intelligent-Tiering** | Unknown/changing | ms | $$ (auto-tiered) |
| **Standard-IA** | Infrequent | ms | $$ |
| **One Zone-IA** | Infrequent, non-critical | ms | $ |
| **Glacier Instant** | Rare, needs instant access | ms | $ |
| **Glacier Flexible** | Archive | minutes to hours | ¢ |
| **Glacier Deep Archive** | Long-term archive | 12-48 hours | ¢¢ |

**Lifecycle policies** automatically transition objects between tiers based on age.

### Performance Optimization

| Technique | How It Works |
|-----------|-------------|
| **Multipart upload** | Split large files into parts, upload in parallel |
| **Transfer acceleration** | Route uploads through CloudFront edge locations |
| **Byte-range fetches** | Download only a portion of an object |
| **Prefix partitioning** | Distribute keys across prefixes to avoid hot partitions |

S3 supports **3,500 PUT/sec and 5,500 GET/sec per prefix**. Distribute keys across prefixes for higher throughput.

## Common Use Cases in System Design

### User-Uploaded Content (Images, Videos)

```
Client → Pre-signed URL from API → Direct upload to S3
                                 → S3 event → Lambda → generate thumbnails → save back to S3
Reads: CloudFront CDN → S3 origin
```

**Pre-signed URLs:** Time-limited URLs that allow clients to upload/download directly to/from S3 without exposing credentials.

### Static Website Hosting

S3 can serve static files (HTML, CSS, JS) directly. Combined with CloudFront for global distribution.

### Data Lake

Store raw data in S3 (CSV, JSON, Parquet). Query with Athena (SQL over S3), process with Spark/EMR.

### Backup & Disaster Recovery

Cross-region replication ensures data survives regional failures. Lifecycle policies move old backups to Glacier.

## Content-Addressable Storage

Objects are identified by the hash of their content (e.g., SHA-256).

```
Key = SHA256(content)
```

**Properties:**
- Automatic deduplication (same content = same key)
- Immutable by definition (changing content changes the key)
- Easy integrity verification

**Used in:** Git (for objects), Docker (for layers), backup systems.

## Key Interview Talking Points

- S3 (or equivalent) is the default for storing unstructured data — images, videos, logs, backups
- Use pre-signed URLs for client-direct uploads/downloads to avoid routing large files through your application
- CDN in front of S3 for read-heavy content (images, static assets)
- Storage tiers optimize cost — mention lifecycle policies for infrequently accessed data
- S3 event notifications trigger serverless processing (thumbnails, transcoding, indexing)
- 11 nines of durability means you can trust S3 as the source of truth for blob data
