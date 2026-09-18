# Database Architecture & Security Protocol: Field Raid Verification Portal

This document details the database schema, storage architecture, anti-tampering verification mechanisms, and API workflow required for the **Field Raid Verification System**.

---

## 1. System Architecture Overview

To enforce strict live-camera verification and prevent gallery file spoofing, media files are decoupled from core relational data:

* **Relational Database (PostgreSQL / Supabase):** Holds officer details, raid metadata, high-precision GPS coordinates, capture methods, and cryptographic image hashes.
* **Object Storage (AWS S3 / Google Cloud Storage):** Holds the raw binary JPEG image files.
* **Client App (Browser / PWA):** Captures camera stream directly into memory buffers and streams payload via authenticated Presigned URLs.

```
┌─────────────────┐       1. Fetch Presigned URL      ┌─────────────────┐
│                 │ ─────────────────────────────────>│  Backend API    │
│  Officer Client │                                   │  (Node/Python)  │
│  (Camera Stream)│ <─────────────────────────────────│                 │
│                 │       2. Return S3 Upload URL     └─────────────────┘
└────────┬────────┘                                            │
         │                                                     │ 4. Save Record
         │ 3. Direct Binary Upload                             ▼
         ▼                                            ┌─────────────────┐
┌─────────────────┐                                   │  PostgreSQL DB  │
│   AWS S3 Bucket │                                   │ (Metadata & Hash│
│ (Evidence Blob) │                                   └─────────────────┘
└─────────────────┘
```

---

## 2. PostgreSQL Schema Design

Below is the complete SQL schema including indexes and integrity constraints.

```sql
-- Enable UUID extension
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";

-- 1. Officers Directory
CREATE TABLE officers (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    badge_number VARCHAR(50) UNIQUE NOT NULL,
    full_name VARCHAR(120) NOT NULL,
    rank VARCHAR(50),
    department VARCHAR(100),
    is_active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

-- 2. Raid Incident Master Records
CREATE TABLE raids (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    raid_code VARCHAR(50) UNIQUE NOT NULL, -- e.g., RAID-883921
    location_name TEXT NOT NULL,
    status VARCHAR(20) DEFAULT 'IN_PROGRESS' CHECK (status IN ('IN_PROGRESS', 'VERIFIED', 'UNDER_REVIEW', 'CLOSED')),
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

-- 3. Core Evidence Verification Table
CREATE TABLE raid_verifications (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    raid_id UUID NOT NULL REFERENCES raids(id) ON DELETE CASCADE,
    officer_id UUID NOT NULL REFERENCES officers(id),
    
    -- Photo Storage Reference
    photo_url TEXT NOT NULL,
    storage_key VARCHAR(255) NOT NULL, -- S3 key path
    
    -- Precision Geolocation
    latitude DECIMAL(10, 8) NOT NULL,
    longitude DECIMAL(11, 8) NOT NULL,
    gps_accuracy_meters DECIMAL(6, 2),
    
    -- Chain of Custody & Verification Flags
    capture_method VARCHAR(30) NOT NULL CHECK (capture_method IN ('WEBRTC_CAMERA', 'HARDWARE_SHUTTER')),
    capture_timestamp TIMESTAMP WITH TIME ZONE NOT NULL,
    upload_timestamp TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    image_hash VARCHAR(64) NOT NULL, -- SHA-256 Checksum
    device_user_agent TEXT,
    notes TEXT,
    
    -- Integrity Check Flag
    is_flagged_for_tampering BOOLEAN DEFAULT FALSE
);

-- Indexes for Fast Query Performance
CREATE INDEX idx_verifications_raid_id ON raid_verifications(raid_id);
CREATE INDEX idx_verifications_officer_id ON raid_verifications(officer_id);
CREATE INDEX idx_verifications_capture_time ON raid_verifications(capture_timestamp);
```

---

## 3. Anti-Tampering & Security Measures

To ensure that pre-captured or edited photos cannot be submitted into the database:

### 1. Direct Buffer Streaming (No File Pickers)
The client JavaScript captures camera frames into an `HTML5 Canvas` or directly into memory array buffers. At no point is the file stored in local storage or file system before upload.

### 2. Client-Side SHA-256 Image Hashing
Before uploading, compute a cryptographic SHA-256 hash of the captured image buffer:
```javascript
async function computeImageHash(imageBlob) {
    const arrayBuffer = await imageBlob.arrayBuffer();
    const hashBuffer = await crypto.subtle.digest('SHA-256', arrayBuffer);
    const hashArray = Array.from(new Uint8Array(hashBuffer));
    return hashArray.map(b => b.toString(16).padStart(2, '0')).join('');
}
```

### 3. Time-Delta Validation
The backend compares `capture_timestamp` (from camera shutter) against `upload_timestamp` (server receipt time). 
* If $\Delta t > 30\text{ seconds}$, flag `is_flagged_for_tampering = TRUE` for manual review.

---

## 4. Sample JSON Payload & API Integration

### POST `/api/v1/verifications`
```json
{
  "raid_id": "c7a9b0a1-4321-4f11-a889-123456789abc",
  "officer_badge": "OFF-8849",
  "latitude": 6.927100,
  "longitude": 79.861200,
  "gps_accuracy": 4.5,
  "capture_method": "WEBRTC_CAMERA",
  "capture_timestamp": "2026-09-18T14:10:00Z",
  "image_hash": "e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855",
  "notes": "Evidence secured at main entrance."
}
```