SnapShare Scaling Plan

1. Assumptions and Daily Active Users (DAU)
* **Registered Users:** 10,000,000
* **Daily Active User (DAU) Percentage:** 10%
* **Daily Active Users (DAU):** 10,000,000 * 10% = **1,000,000 DAU**
* **User Behavior per Active User / Day:**
  * Uploads: 1 photo / day
  * Feed Views: 50 feed pages / day
* **File Sizes:**
  * Original Photo: 2 MB
  * Thumbnail Photo: 50 KB (0.05 MB)
* **Time Assumptions:** 1 day = 86,400 seconds (rounded to ~100,000 seconds for quick estimates, but exact math using 86,400 s is provided below).



2. Traffic and Storage Calculations

Uploads Per Second
* **Total Daily Uploads:** 1,000,000 photos / day
* **Average Uploads / Sec:** 1,000,000 / 86,400 ≈ **11.57 uploads/sec**
* **Peak Uploads / Sec (5× multiplier):** 11.57 * 5 ≈ **57.87 uploads/sec**

Feed Views Per Second
* **Total Daily Feed Views:** 1,000,000 DAU * 50 views/day = 50,000,000 views / day
* **Average Feed Views / Sec:** 50,000,000 / 86,400 ≈ **578.70 views/sec**
* **Peak Feed Views / Sec (5× multiplier):** 578.70 * 5 ≈ **2,893.52 views/sec**

Photo Storage Per Year
* **Storage per photo upload:** 2 MB (original) + 0.05 MB (thumbnail) = 2.05 MB
* **Daily Storage Added:** 1,000,000 * 2.05 MB = 2,050,000 MB = 2,050 GB = **2.05 TB / day**
* **Yearly Storage Added:** 2.05 TB/day * 365 days ≈ **748.25 TB / year** (or ~730 TB if calculating using 1024 conversion factors).


3. Read-Heavy vs. Write-Heavy System
SnapShare is a **read-heavy system**. 

* **Ratio:** There are 50 feed views for every 1 photo upload (a 50:1 read-to-write ratio).
* **Design Implications:** Because read operations dominate traffic, the system design must prioritize fast content retrieval over fast write processing. This means aggressive caching strategies (using a CDN for static images and in-memory caches for feed metadata/relationships), read-replicas for database scaling, and asynchronous offloading for background tasks (like processing image uploads).


4. Binary Media Storage Strategy
Photos should **not** be stored directly inside a relational database as BLOBs (Binary Large Objects) because large files bloat database storage, consume high memory during operations, make backups/restores slow, and lock database connections while transferring raw bytes.

Instead, photos should be stored in an **Object Storage Service** (such as AWS S3 or Google Cloud Storage), while the relational database only stores lightweight metadata (such as image file paths/URLs, upload timestamps, owner IDs, and photo tags).


5. Architecture Diagram

```text
                                +-------------------+
                                |    Client App     |
                                +---------+---------+
                                          |
                                          v
                                +-------------------+
                                |   CDN (CloudFront)|
                                +---------+---------+
                                          |
                                          v
                                +-------------------+
                                |   Load Balancer   |
                                +---------+---------+
                                          |
                                          v
                                +-------------------+
                                |    App Servers    |
                                +----+----+----+----+
                                     |    |    |
           +-------------------------+    |    +------------------------+
           |                              |                             |
           v                              v                             v
+--------------------+         +--------------------+        +--------------------+
| In-Memory Cache    |         | Object Storage     |        | Message Queue      |
| (Redis/Memcached)  |         | (Original Photos)  |        | (SQS / RabbitMQ)   |
+--------------------+         +--------------------+        +----------+---------+
                                          ^                             |
                                          |                             v
                                          |                  +--------------------+
                                          +------------------|  Thumbnail Worker  |
                                 (Writes Thumbnails)         +--------------------+
                                          |
                                          v
                               +--------------------+
                               | Main Database      |
                               | (Writes & Master)  |
                               +----------+---------+
                                          |
                                  (Replication)
                                          |
                                          v
                               +--------------------+
                               | DB Read Replica    |
                               | (Read Traffic)     |
                               +--------------------+