# NightBreath Guardian – System Design Document

## Technical Design Specification

**Version:** 1.0  
**Date:** February 14, 2026  
**Project Type:** Professional Healthcare project 
**Domain:** IoT, AI/ML, Healthcare Technology

---

## 1. System Architecture

### 1.1 Architecture Overview

The NightBreath Guardian system follows a three-tier architecture: Edge Layer (ESP32 + Sensors), Cloud Layer (Backend + AI Processing), and Application Layer (Web/Mobile Dashboard).

```
┌─────────────────────────────────────────────────────────────┐
│                   APPLICATION LAYER                          │
│  ┌──────────────────┐         ┌──────────────────┐          │
│  │  Web Dashboard   │         │  Mobile App      │          │
│  │  (React.js)      │         │  (Flutter)       │          │
│  └──────────────────┘         └──────────────────┘          │
└─────────────────────────────────────────────────────────────┘
                            ▲
                            │ HTTPS/WebSocket
                            ▼
┌─────────────────────────────────────────────────────────────┐
│                      CLOUD LAYER                             │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐      │
│  │ API Gateway  │  │  ML Service  │  │ Notification │      │
│  │  (FastAPI)   │  │  (PyTorch)   │  │   Service    │      │
│  └──────────────┘  └──────────────┘  └──────────────┘      │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐      │
│  │ PostgreSQL   │  │    Redis     │  │   AWS S3     │      │
│  └──────────────┘  └──────────────┘  └──────────────┘      │
└─────────────────────────────────────────────────────────────┘
                            ▲
                            │ MQTT over TLS
                            ▼
┌─────────────────────────────────────────────────────────────┐
│                       EDGE LAYER                             │
│  ┌─────────────────────────────────────────────────────┐    │
│  │  ESP32 Microcontroller                              │    │
│  │  - Audio Processing                                 │    │
│  │  - Feature Extraction                               │    │
│  │  - Edge AI (TFLite)                                 │    │
│  └─────────────────────────────────────────────────────┘    │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐      │
│  │ INMP441 MEMS │  │   DHT22      │  │   Battery    │      │
│  │ Microphone   │  │ Temp/Humidity│  │   Backup     │      │
│  └──────────────┘  └──────────────┘  └──────────────┘      │
└─────────────────────────────────────────────────────────────┘
```

### 1.2 Data Flow

```
Sensor → Edge Processing → Cloud AI → Classification → Alert → Dashboard
  (1)         (2)              (3)          (4)         (5)       (6)

1. Sensors capture audio at 16kHz, environmental data at 0.5Hz
2. ESP32 extracts features (MFCC, breathing rate), runs edge AI
3. Anomalies sent to cloud for deep learning analysis
4. Events classified as Normal/Warning/Critical
5. Alerts generated and sent via push/SMS/email
6. Real-time updates displayed on dashboard
```

**Timing:** Total latency from detection to alert: <5 seconds

---

## 2. High-Level Design

### 2.1 System Workflow

**Phase 1: Baseline Learning (7-14 days)**
- Collect normal breathing patterns during sleep
- Train personalized AI models for each user
- Establish baseline metrics (breathing rate, regularity)

**Phase 2: Real-Time Monitoring (Continuous)**
```
Audio Capture (ESP32)
    ↓
Preprocessing (Filtering, FFT)
    ↓
Feature Extraction (MFCC, Breathing Rate)
    ↓
Edge AI Inference (Autoencoder)
    ↓
Normal? → Log locally
Anomaly? → Send to Cloud
    ↓
Cloud ML Analysis (LSTM + CNN)
    ↓
Event Classification
    ↓
Score < 0.65: Normal (no alert)
Score 0.65-0.85: Warning (notify primary caregiver)
Score > 0.85: Critical (multi-channel alert)
    ↓
Notification Delivery
    ↓
Dashboard Update
```

### 2.2 Event Classification Logic

| Severity | Score Range | Conditions | Action |
|----------|-------------|------------|--------|
| Normal | < 0.65 | Within baseline ±30% | Log only |
| Warning | 0.65-0.85 | Deviation 30-50%, pauses 10-20s | Alert primary caregiver |
| Critical | ≥ 0.85 | Deviation >50%, cessation >20s | Multi-channel alert all caregivers |

---

## 3. AI Model Overview

### 3.1 Feature Extraction

**Audio Features:**
- **MFCC (Mel-Frequency Cepstral Coefficients):** 13 coefficients extracted from 5-second windows
- **Zero-Crossing Rate:** Indicates signal frequency content
- **Spectral Centroid:** Center of mass of frequency spectrum
- **RMSE (Root Mean Square Energy):** Overall signal energy

**Breathing Features:**
- **Respiration Rate:** Breaths per minute (normal: 12-20 BPM)
- **Breathing Amplitude Variance:** Regularity indicator
- **Inter-Breath Interval:** Time between breathing cycles
- **Apnea Detection:** Periods with no breathing >10 seconds

**Processing Pipeline:**
```
Raw Audio (16kHz) 
  → Butterworth Filter (200-2000Hz)
  → FFT (1024 points)
  → MFCC Extraction (13 coefficients)
  → Statistical Features (ZCR, Spectral)
  → Feature Vector (18 dimensions)
```

### 3.2 AI Model Architecture

**Edge AI: Autoencoder (ESP32)**
```python
# Lightweight model for edge deployment
Input: 13 MFCCs
  ↓
Encoder: [13 → 32 → 16 → 4] (Latent space)
  ↓
Decoder: [4 → 16 → 32 → 13] (Reconstruction)
  ↓
Reconstruction Error > Threshold → Anomaly

Model Size: <200 KB (INT8 quantized)
Inference Time: 50-80ms
```

**Cloud AI: LSTM + CNN Ensemble**

**LSTM Model (Temporal Analysis):**
```python
Input: (24 time steps, 18 features) # 60-second context
  ↓
Bidirectional LSTM: 64 units × 2 layers
  ↓
LSTM: 32 units × 1 layer
  ↓
Attention Mechanism
  ↓
Fully Connected: [32 → 16 → 1]
  ↓
Output: Anomaly Score (0-1)
```

**CNN Model (Acoustic Event Detection):**
```python
Input: (13 MFCCs, 500 time frames) # Spectrogram
  ↓
Conv2D: 32 filters (3×3) → MaxPool
  ↓
Conv2D: 64 filters (3×3) → MaxPool
  ↓
Conv2D: 128 filters (3×3) → MaxPool
  ↓
Flatten → Dense: [256 → 64 → 1]
  ↓
Output: Anomaly Score (0-1)
```

**Ensemble Decision:**
```
Final Score = 0.6 × LSTM_score + 0.4 × CNN_score
```

### 3.3 Edge vs Cloud Processing

| Aspect | Edge (ESP32) | Cloud (AWS) |
|--------|--------------|-------------|
| **Model** | Autoencoder (200KB) | LSTM + CNN (13MB) |
| **Latency** | 80ms | 800ms |
| **Accuracy** | 75-80% | 92-95% |
| **Power** | 180mA | N/A |
| **Privacy** | High (no data leaves device) | Medium (features only) |
| **Cost** | One-time hardware | $3-5/user/month |

**Decision Rationale:**
- Edge AI filters 70% of normal events locally (privacy + cost savings)
- Cloud AI handles complex analysis for anomalies (accuracy)
- Hybrid approach balances latency, accuracy, and cost

---

## 4. Component Breakdown

### 4.1 Hardware Components

**ESP32-WROOM-32D Microcontroller**
- Dual-core Xtensa LX6 @ 240MHz
- 520KB SRAM, 4MB Flash
- Wi-Fi 802.11 b/g/n
- I2S, I2C interfaces

**INMP441 MEMS Microphone**
- I2S digital output
- Frequency response: 60Hz - 15kHz
- SNR: 61dB
- Omnidirectional pickup

**DHT22 Temperature/Humidity Sensor**
- Temperature: -40°C to 80°C (±0.5°C)
- Humidity: 0-100% RH (±2%)
- I2C interface

**Power System**
- Primary: 5V/2A adapter
- Backup: 18650 Li-ion battery (3000mAh)
- Runtime: 48+ hours on battery

### 4.2 Backend Services

**API Gateway (FastAPI)**
- RESTful API endpoints
- WebSocket for real-time updates
- JWT authentication
- Request validation

**ML Service (Python)**
- PyTorch model inference
- Baseline model management
- Model retraining pipeline
- Feature preprocessing

**Notification Service**
- Push notifications (Firebase Cloud Messaging)
- SMS (Twilio API)
- Email (AWS SES)
- Multi-channel delivery

### 4.3 Database (PostgreSQL)

**Core Tables:**
- `users`: User profiles and medical history
- `devices`: ESP32 device registry
- `sleep_sessions`: Sleep monitoring sessions
- `sensor_data`: Time-series sensor readings
- `alerts`: Alert history and acknowledgments
- `notifications`: Delivery tracking

### 4.4 Web/Mobile Dashboard

**Web Dashboard (React.js)**
- Real-time monitoring view
- Alert history and management
- Data visualization (charts)
- User settings and preferences

**Mobile App (Flutter)**
- Cross-platform (iOS/Android)
- Push notification handling
- Offline capability
- Biometric authentication

---

## 5. Technology Stack

### 5.1 Embedded Layer

**Language:** C/C++ (ESP-IDF framework)

**Libraries:**
- ESP-DSP: Signal processing (FFT, filters)
- TensorFlow Lite Micro: On-device ML
- ESP-MQTT: MQTT client with TLS
- cJSON: JSON serialization

**Rationale:** Official ESP32 framework with optimized hardware support and FreeRTOS integration.

### 5.2 Backend Stack

**Framework:** FastAPI (Python 3.11)

**Rationale:** High performance async framework, automatic API documentation, type safety with Pydantic.

**Alternative:** Node.js (considered but Python preferred for ML integration)

### 5.3 AI/ML Stack

**Framework:** PyTorch 2.0

**Libraries:**
- librosa: Audio feature extraction
- scikit-learn: Preprocessing, classical ML
- numpy/scipy: Numerical computations

**Rationale:** Dynamic computation graphs, excellent for research and production, strong ecosystem.

### 5.4 Database

**Primary:** PostgreSQL 14

**Rationale:** ACID compliance, JSON support, time-series optimization, mature ecosystem.

**Cache:** Redis 7.0 (session management, real-time pub/sub)

### 5.5 Cloud Infrastructure

**Provider:** AWS (Amazon Web Services)

**Services:**
- EC2: Compute instances
- RDS: Managed PostgreSQL
- S3: Object storage (raw data, models)
- IoT Core: MQTT broker
- SES: Email delivery
- CloudWatch: Monitoring

**Rationale:** Comprehensive IoT services, mature ML infrastructure, global availability.

### 5.6 Frontend

**Web:** React.js 18 + Redux Toolkit + Material-UI

**Mobile:** Flutter 3.0 (single codebase for iOS/Android)

**Charts:** Chart.js, D3.js

---

## 6. Database Design

### 6.1 Schema Design

**Users Table**
```sql
CREATE TABLE users (
    user_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email VARCHAR(255) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    full_name VARCHAR(255) NOT NULL,
    date_of_birth DATE,
    medical_conditions TEXT[],
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_users_email ON users(email);
```

**Devices Table**
```sql
CREATE TABLE devices (
    device_id VARCHAR(50) PRIMARY KEY,  -- ESP32 MAC address
    user_id UUID REFERENCES users(user_id) ON DELETE CASCADE,
    device_name VARCHAR(100),
    firmware_version VARCHAR(20),
    last_seen TIMESTAMP,
    status VARCHAR(20) DEFAULT 'active',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_devices_user ON devices(user_id);
```

**Sleep Sessions Table**
```sql
CREATE TABLE sleep_sessions (
    session_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID REFERENCES users(user_id) ON DELETE CASCADE,
    device_id VARCHAR(50) REFERENCES devices(device_id),
    start_time TIMESTAMP NOT NULL,
    end_time TIMESTAMP,
    avg_breathing_rate FLOAT,
    sleep_quality_score FLOAT CHECK (sleep_quality_score BETWEEN 0 AND 1),
    total_events INTEGER DEFAULT 0,
    critical_events INTEGER DEFAULT 0,
    warning_events INTEGER DEFAULT 0,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_sessions_user_time ON sleep_sessions(user_id, start_time DESC);
```

**Alerts Table**
```sql
CREATE TABLE alerts (
    alert_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID REFERENCES users(user_id) ON DELETE CASCADE,
    session_id UUID REFERENCES sleep_sessions(session_id),
    severity VARCHAR(20) NOT NULL CHECK (severity IN ('normal', 'warning', 'critical')),
    event_type VARCHAR(50),
    confidence_score FLOAT CHECK (confidence_score BETWEEN 0 AND 1),
    breathing_rate FLOAT,
    description TEXT,
    acknowledged BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_alerts_user_time ON alerts(user_id, created_at DESC);
CREATE INDEX idx_alerts_severity ON alerts(severity, created_at DESC);
```

**Sensor Data Table (Time-Series)**
```sql
CREATE TABLE sensor_data (
    data_id BIGSERIAL PRIMARY KEY,
    session_id UUID REFERENCES sleep_sessions(session_id) ON DELETE CASCADE,
    timestamp TIMESTAMP NOT NULL,
    breathing_rate FLOAT,
    temperature FLOAT,
    humidity FLOAT,
    mfcc_features FLOAT[13],
    anomaly_score FLOAT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_sensor_timestamp ON sensor_data(timestamp DESC);
CREATE INDEX idx_sensor_session_time ON sensor_data(session_id, timestamp DESC);
```

### 6.2 Relationships

```
users (1) ──→ (N) devices
users (1) ──→ (N) sleep_sessions
users (1) ──→ (N) alerts
sleep_sessions (1) ──→ (N) sensor_data
sleep_sessions (1) ──→ (N) alerts
```

---

## 7. API Design

### 7.1 Authentication Endpoints

**POST /api/v1/auth/login**
```json
Request:
{
  "email": "user@example.com",
  "password": "SecurePass123!"
}

Response (200 OK):
{
  "access_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "refresh_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "token_type": "bearer",
  "expires_in": 3600,
  "user": {
    "user_id": "550e8400-e29b-41d4-a716-446655440000",
    "email": "user@example.com",
    "full_name": "John Doe"
  }
}
```

### 7.2 Sensor Data Ingestion

**POST /api/v1/monitoring/ingest**
```json
Request:
Headers: Authorization: Bearer <device_token>

{
  "device_id": "AA:BB:CC:DD:EE:FF",
  "timestamp": 1708000000,
  "features": {
    "mfcc": [1.2, -0.5, 0.8, 0.3, -0.2, 0.9, -0.1, 0.4, 0.6, -0.3, 0.7, 0.2, -0.4],
    "breathing_rate": 16.5,
    "zero_crossing_rate": 0.12,
    "spectral_centroid": 850.3,
    "rmse": 0.045,
    "edge_anomaly_score": 0.18
  },
  "context": {
    "temperature": 22.5,
    "humidity": 45.0
  }
}

Response (200 OK):
{
  "status": "processed",
  "classification": "normal",
  "anomaly_score": 0.23,
  "confidence": 0.92,
  "alert_generated": false
}
```

### 7.3 Alert Retrieval

**GET /api/v1/alerts**
```json
Query Parameters:
  - user_id: UUID (required)
  - severity: string (optional)
  - limit: integer (default: 50)

Response (200 OK):
{
  "total": 127,
  "alerts": [
    {
      "alert_id": "660e8400-e29b-41d4-a716-446655440000",
      "severity": "warning",
      "event_type": "irregular_breathing",
      "description": "Breathing rate deviation: 25 BPM (baseline: 16 BPM)",
      "confidence_score": 0.88,
      "breathing_rate": 25.0,
      "acknowledged": false,
      "created_at": "2026-02-14T03:15:22Z"
    }
  ]
}
```

**PUT /api/v1/alerts/{alert_id}/acknowledge**
```json
Request:
Headers: Authorization: Bearer <caregiver_token>

{
  "notes": "Called patient, they were okay."
}

Response (200 OK):
{
  "alert_id": "660e8400-e29b-41d4-a716-446655440000",
  "acknowledged": true,
  "acknowledged_at": "2026-02-14T03:18:00Z"
}
```

### 7.4 Real-Time Monitoring

**GET /api/v1/monitoring/current**
```json
Query Parameters:
  - user_id: UUID (required)

Response (200 OK):
{
  "user_id": "550e8400-e29b-41d4-a716-446655440000",
  "session_id": "770e8400-e29b-41d4-a716-446655440000",
  "status": "monitoring",
  "current_breathing_rate": 16.2,
  "session_start": "2026-02-14T22:30:00Z",
  "duration_minutes": 285,
  "events_count": {
    "normal": 342,
    "warning": 2,
    "critical": 0
  },
  "last_update": "2026-02-14T03:15:00Z"
}
```

---

## 8. Security Design

### 8.1 Data Encryption

**In Transit (TLS 1.3)**
- All communication encrypted with TLS 1.3
- ESP32 to Cloud: MQTT over TLS (port 8883)
- Client to API: HTTPS only
- Certificate-based device authentication (X.509)

**At Rest (AES-256)**
- Database encryption via AWS RDS (KMS-managed keys)
- S3 object encryption (server-side)
- Sensitive fields additionally encrypted at application level

### 8.2 Authentication & Authorization

**JWT (JSON Web Tokens)**
```python
# Token structure
{
  "sub": "user_id",
  "email": "user@example.com",
  "role": "caregiver",
  "exp": 1708003600,  # Expiration timestamp
  "iat": 1708000000   # Issued at timestamp
}

# Token generation
def create_access_token(user_id: str, role: str) -> str:
    payload = {
        'sub': user_id,
        'role': role,
        'exp': datetime.utcnow() + timedelta(hours=1)
    }
    return jwt.encode(payload, SECRET_KEY, algorithm='RS256')
```

**Role-Based Access Control (RBAC)**
- Monitored User: View own data, manage devices
- Primary Caregiver: View data, acknowledge alerts, configure notifications
- Secondary Caregiver: View data, acknowledge alerts
- Admin: Full system access

### 8.3 Device Security

**Secure Device Provisioning**
1. Device generates unique key pair during manufacturing
2. User scans QR code to register device
3. API generates X.509 certificate for device
4. Device uses certificate for MQTT authentication

**Firmware Security**
- OTA updates signed with RSA private key
- Device verifies signature before installation
- Rollback protection on failed updates
- Secure boot enabled on ESP32

### 8.4 Privacy Protection

**Data Minimization**
- Raw audio never transmitted or stored
- Only mathematical features (MFCC) sent to cloud
- No identifiable voice content captured

**Access Controls**
- User data isolated by user_id
- Caregivers only access authorized users
- Audit logging for all data access

---

## 9. Scalability Considerations

### 9.1 Handling Multiple Devices

**Edge Device Management**
- Each device operates independently
- Local buffering during network outages (up to 24 hours)
- Automatic reconnection and data sync
- Device fleet management via AWS IoT Core

**Load Distribution**
- MQTT broker (AWS IoT Core) handles 1M+ concurrent connections
- Message routing via IoT Rules Engine
- SQS queues decouple ingestion from processing

### 9.2 Cloud Scaling Strategy

**Horizontal Scaling**
```
┌─────────────────────────────────────────────┐
│     Application Load Balancer (ALB)         │
└─────────────────┬───────────────────────────┘
                  │
        ┌─────────┼─────────┐
        ▼         ▼         ▼
    ┌──────┐  ┌──────┐  ┌──────┐
    │ API  │  │ API  │  │ API  │  Auto-scaling
    │ Pod 1│  │ Pod 2│  │ Pod N│  (2-20 instances)
    └──────┘  └──────┘  └──────┘
```

**Auto-Scaling Rules**
- Scale up: CPU > 70% for 5 minutes
- Scale down: CPU < 30% for 10 minutes
- Min instances: 2 (high availability)
- Max instances: 20 (cost control)

**Database Scaling**
- Read replicas for analytics queries
- Connection pooling (max 20 connections per instance)
- Query optimization with indexes
- Partitioning for time-series data (monthly partitions)

**Caching Strategy**
- Redis for user baselines (1-hour TTL)
- Session data caching
- API response caching for static data
- Cache invalidation on updates

### 9.3 Performance Optimization

**Target Metrics**
- API latency: p95 < 200ms
- ML inference: < 1 second
- Alert delivery: < 2 seconds
- Dashboard load: < 3 seconds
- System uptime: 99.5%

**Optimization Techniques**
- Async processing for non-critical tasks
- Batch processing for model training
- CDN for static assets
- Database query optimization
- Connection pooling

### 9.4 Cost Optimization

**Estimated Monthly Cost (per 1000 users)**
- EC2 instances: $300-400
- RDS PostgreSQL: $200-300
- S3 storage: $50-100
- Data transfer: $100-150
- IoT Core: $150-200
- **Total: $800-1150 ($0.80-1.15 per user)**

**Cost Reduction Strategies**
- Edge AI reduces cloud processing by 70%
- S3 lifecycle policies (archive old data to Glacier)
- Reserved instances for predictable workload
- Spot instances for batch processing

---

## 10. Deployment Architecture

### 10.1 Production Deployment

```
┌─────────────────────────────────────────────────────────┐
│                    AWS CLOUD                             │
│                                                          │
│  ┌────────────┐      ┌────────────┐                    │
│  │    ALB     │─────→│  ECS/EC2   │ (API Service)      │
│  │            │      │  Cluster   │                     │
│  └────────────┘      └────────────┘                     │
│                                                          │
│  ┌────────────┐      ┌────────────┐                    │
│  │ RDS        │      │ ElastiCache│                     │
│  │ PostgreSQL │      │   Redis    │                     │
│  └────────────┘      └────────────┘                     │
│                                                          │
│  ┌────────────┐      ┌────────────┐                    │
│  │ AWS IoT    │      │    S3      │                     │
│  │   Core     │      │  Storage   │                     │
│  └────────────┘      └────────────┘                     │
└─────────────────────────────────────────────────────────┘
```

### 10.2 CI/CD Pipeline

**GitHub Actions Workflow**
1. Code push to repository
2. Run automated tests (pytest, unit tests)
3. Build Docker images
4. Push to AWS ECR (Elastic Container Registry)
5. Deploy to ECS (rolling update)
6. Health check verification
7. Rollback on failure

### 10.3 Monitoring & Logging

**CloudWatch Metrics**
- API request rate and latency
- ML inference time
- Alert generation rate
- Device online/offline status
- Database performance
- Error rates

**Alerting**
- Critical: System downtime, database failures
- Warning: High latency, increased error rates
- Info: Deployment events, scaling activities

---

## Document Approval

This design document provides the technical foundation for the NightBreath Guardian system implementation.

**Prepared By:** Team Hackaholics   
**Date:** February 14, 2026  
**Version:** 1.0

---

**End of Design Document**
