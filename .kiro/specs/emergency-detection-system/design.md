# Design Document: Emergency Detection System

## Overview

The Emergency Detection System is a real-time video analytics platform that monitors CCTV feeds to detect emergency situations and automatically alert appropriate authorities. The system employs computer vision and machine learning models to identify three categories of emergencies: physical violence, medical emergencies (collapses), and fire incidents.

The architecture follows a pipeline pattern with four main stages:
1. **Video Ingestion**: Continuous streaming from multiple CCTV sources
2. **Detection & Analysis**: ML-based event detection with confidence scoring
3. **Authority Routing**: Geographic and incident-type based routing logic
4. **Alert Dispatch**: Multi-channel notification with acknowledgment tracking

The system is designed for high availability with redundancy at critical points, sub-2-second detection latency, and comprehensive audit logging for compliance and continuous improvement.

## Architecture

### System Components

```
┌─────────────────┐
│  CCTV Feeds     │
│  (Multiple)     │
└────────┬────────┘
         │
         ▼
┌────────────────────────────────────────────────┐
│         Video Ingestion Layer                  │
│  ┌──────────────┐  ┌──────────────┐            │
│  │ Feed Manager │  │ Load Balancer│            │
│  └──────────────┘  └──────────────┘            │
└────────┬───────────────────────────────────────┘
         │
         ▼
┌────────────────────────────────────────────────┐
│         Detection & Analysis Layer             │
│  ┌──────────────┐  ┌──────────────┐            │
│  │   Violence   │  │   Collapse   │            │
│  │   Detector   │  │   Detector   │            │
│  └──────────────┘  └──────────────┘            │
│  ┌──────────────┐  ┌──────────────┐            │
│  │     Fire     │  │  Confidence  │            │
│  │   Detector   │  │   Scorer     │            │
│  └──────────────┘  └──────────────┘            │
└────────┬───────────────────────────────────────┘
         │
         ▼
┌────────────────────────────────────────────────┐
│         Event Processing Layer                 │
│  ┌──────────────┐  ┌──────────────┐            │
│  │    Event     │  │   Evidence   │            │
│  │   Manager    │  │   Capture    │            │
│  └──────────────┘  └──────────────┘            │
└────────┬───────────────────────────────────────┘
         │
         ▼
┌────────────────────────────────────────────────┐
│         Authority Routing Layer                │
│  ┌──────────────┐  ┌──────────────┐            │
│  │  Authority   │  │   Location   │            │
│  │   Locator    │  │   Service    │            │
│  └──────────────┘  └──────────────┘            │
└────────┬───────────────────────────────────────┘
         │
         ▼
┌────────────────────────────────────────────────┐
│         Alert Dispatch Layer                   │
│  ┌──────────────┐  ┌──────────────┐            │
│  │    Alert     │  │ Multi-Channel│            │
│  │  Generator   │  │   Sender     │            │
│  └──────────────┘  └──────────────┘            │
│  ┌──────────────┐                              │
│  │Acknowledgment│                              │
│  │   Tracker    │                              │
│  └──────────────┘                              │
└────────────────────────────────────────────────┘
         │
         ▼
┌────────────────────────────────────────────────┐
│         Storage & Monitoring Layer             │
│  │   Database   │  │  Video Store │            │
│  └──────────────┘  └──────────────┘            │
│  ┌──────────────┐  ┌──────────────┐            │
│  ┌──────────────┐  ┌──────────────┐            │
│  │  Dashboard   │  │   Metrics    │            │
│  └──────────────┘  └──────────────┘            │
└────────────────────────────────────────────────┘
```

### Deployment Architecture

The system is designed for cloud deployment with the following characteristics:

- **Video Ingestion**: Containerized services with auto-scaling based on feed count
- **Detection Layer**: GPU-enabled instances for ML inference, horizontally scalable
- **Event Processing**: Stateless microservices with message queue buffering
- **Storage**: Object storage for video evidence, relational database for metadata
- **High Availability**: Multi-zone deployment with automatic failover

## Components and Interfaces

### 1. Feed Manager

**Responsibility**: Manages connections to CCTV feeds, handles reconnection logic, and distributes feeds to detector instances.

**Interface**:
```
FeedManager:
  - registerFeed(feedId: string, streamUrl: string, location: Location): Result
  - unregisterFeed(feedId: string): Result
  - getFeedStatus(feedId: string): FeedStatus
  - listActiveFeeds(): List<FeedInfo>
  
FeedStatus:
  - feedId: string
  - status: enum(connected, disconnected, error)
  - lastFrameTime: timestamp
  - assignedDetector: string
```

**Behavior**:
- Maintains persistent connections to CCTV streams
- Implements exponential backoff for reconnection (1s, 2s, 4s, up to 60s)
- Monitors frame rate and reports degradation
- Redistributes feeds when detector instances fail

### 2. Video Analyzer (Detector Modules)

**Responsibility**: Processes video frames using ML models to detect emergency events.

**Interface**:
```
VideoAnalyzer:
  - processFrame(feedId: string, frame: VideoFrame, timestamp: timestamp): DetectionResult
  - updateModel(detectorType: IncidentType, modelPath: string): Result
  - getDetectorStats(detectorType: IncidentType): DetectorStats

DetectionResult:
  - detected: boolean
  - incidentType: enum(violence, collapse, fire)
  - confidence: float [0.0, 1.0]
  - boundingBoxes: List<BoundingBox>
  - severity: enum(low, medium, high, critical)

DetectorStats:
  - totalFramesProcessed: int
  - detectionsCount: int
  - averageLatency: duration
  - falsePositiveRate: float
```

**Behavior**:
- Each detector type (violence, collapse, fire) runs as an independent module
- Maintains a sliding window buffer of recent frames for context
- Applies confidence thresholds configured per camera
- Outputs detection events only when confidence exceeds threshold

**ML Model Specifications**:
- Violence Detector: Action recognition model (e.g., I3D, SlowFast) trained on violence datasets
- Collapse Detector: Pose estimation + fall detection model (e.g., OpenPose + LSTM)
- Fire Detector: CNN-based fire/smoke detection model with color and motion features

### 3. Event Manager

**Responsibility**: Creates, tracks, and manages emergency event lifecycle.

**Interface**:
```
EventManager:
  - createEvent(detection: DetectionResult, feedId: string, location: Location): EmergencyEvent
  - updateEventStatus(eventId: string, status: EventStatus): Result
  - getEvent(eventId: string): EmergencyEvent
  - queryEvents(filters: EventFilters): List<EmergencyEvent>
  - markFalsePositive(eventId: string, operatorId: string, reason: string): Result

EmergencyEvent:
  - eventId: string
  - incidentType: IncidentType
  - severity: Severity
  - location: Location
  - timestamp: timestamp
  - status: enum(pending, acknowledged, resolved, false_positive)
  - videoEvidence: VideoClip
  - assignedAuthority: AuthorityInfo
  - acknowledgmentTime: optional<timestamp>
  - responderId: optional<string>

EventStatus:
  - status: enum(pending, acknowledged, resolved, false_positive)
  - updatedBy: string
  - updateTime: timestamp
```

**Behavior**:
- Generates unique event IDs
- Triggers evidence capture immediately upon event creation
- Maintains event state machine: pending → acknowledged → resolved
- Supports operator review and false positive marking

### 4. Evidence Capture

**Responsibility**: Captures and stores video clips surrounding detected events.

**Interface**:
```
EvidenceCapture:
  - captureEvidence(feedId: string, eventTime: timestamp, preBuffer: duration, postBuffer: duration): VideoClip
  - getEvidence(eventId: string): VideoClip
  - deleteEvidence(eventId: string): Result

VideoClip:
  - clipId: string
  - feedId: string
  - startTime: timestamp
  - endTime: timestamp
  - storageUrl: string
  - encrypted: boolean
  - accessLog: List<AccessRecord>
```

**Behavior**:
- Maintains a rolling buffer of recent frames per feed (minimum 10 seconds)
- Captures 10 seconds before and after detection (configurable)
- For ongoing events (fire), continues capture until event resolved
- Encrypts video data using AES-256
- Implements automatic deletion after retention period (90 days for events, 24 hours for non-events)

### 5. Authority Locator

**Responsibility**: Identifies the nearest appropriate emergency service based on incident type and location.

**Interface**:
```
AuthorityLocator:
  - findNearestAuthority(incidentType: IncidentType, location: Location): AuthorityInfo
  - updateAuthorityDatabase(authorities: List<AuthorityInfo>): Result
  - getAuthorityInfo(authorityId: string): AuthorityInfo

AuthorityInfo:
  - authorityId: string
  - type: enum(police, hospital, fire_brigade)
  - name: string
  - location: Location
  - contactChannels: List<ContactChannel>
  - backupContacts: List<ContactChannel>
  - serviceArea: GeographicBoundary
  - responseTime: duration

ContactChannel:
  - type: enum(sms, email, webhook, phone)
  - address: string
  - priority: int
```

**Behavior**:
- Maps incident types to authority types: violence→police, collapse→hospital, fire→fire_brigade
- Uses geographic distance calculation (haversine formula for lat/long)
- Maintains cached authority database with periodic refresh
- Falls back to backup contacts if primary unavailable
- Considers service area boundaries and response time estimates

### 6. Location Service

**Responsibility**: Determines precise physical location of incidents from camera metadata.

**Interface**:
```
LocationService:
  - getLocation(feedId: string): Location
  - updateCameraLocation(feedId: string, location: Location): Result
  - validateLocation(location: Location): ValidationResult

Location:
  - buildingName: string
  - floor: optional<string>
  - cameraId: string
  - coordinates: optional<GeoCoordinates>
  - address: string
  - mapReference: optional<string>

GeoCoordinates:
  - latitude: float
  - longitude: float
```

**Behavior**:
- Maintains mapping of camera IDs to physical locations
- Validates location completeness before alert generation
- Supports both indoor (building/floor) and outdoor (GPS) locations

### 7. Alert Generator

**Responsibility**: Creates formatted alert messages with all relevant incident information.

**Interface**:
```
AlertGenerator:
  - generateAlert(event: EmergencyEvent, authority: AuthorityInfo): Alert
  - formatForChannel(alert: Alert, channel: ChannelType): FormattedMessage

Alert:
  - alertId: string
  - eventId: string
  - incidentType: IncidentType
  - severity: Severity
  - location: Location
  - timestamp: timestamp
  - description: string
  - videoEvidenceUrl: string
  - recipientAuthority: AuthorityInfo

FormattedMessage:
  - channel: ChannelType
  - content: string
  - metadata: Map<string, string>
```

**Behavior**:
- Generates human-readable incident descriptions
- Includes secure links to video evidence
- Formats messages appropriately for each channel (SMS: concise, Email: detailed)
- Includes callback URLs for acknowledgment

### 8. Multi-Channel Sender

**Responsibility**: Dispatches alerts through multiple communication channels.

**Interface**:
```
MultiChannelSender:
  - sendAlert(alert: Alert, channels: List<ContactChannel>): SendResult
  - retryFailedSend(alertId: string, channel: ContactChannel): Result
  - getSendStatus(alertId: string): SendStatus

SendResult:
  - alertId: string
  - channelResults: Map<ChannelType, ChannelSendResult>
  - overallSuccess: boolean

ChannelSendResult:
  - channel: ChannelType
  - success: boolean
  - sentTime: timestamp
  - error: optional<string>
  - messageId: optional<string>
```

**Behavior**:
- Sends alerts through all configured channels simultaneously
- Implements retry logic with exponential backoff for failed sends
- Tracks delivery status per channel
- Logs all send attempts for audit trail

### 9. Acknowledgment Tracker

**Responsibility**: Monitors alert acknowledgments and triggers escalation if needed.

**Interface**:
```
AcknowledgmentTracker:
  - recordAcknowledgment(alertId: string, responderId: string, timestamp: timestamp): Result
  - checkPendingAlerts(): List<PendingAlert>
  - escalateAlert(alertId: string): Result

PendingAlert:
  - alertId: string
  - eventId: string
  - sentTime: timestamp
  - timeSinceSent: duration
  - escalationNeeded: boolean
```

**Behavior**:
- Monitors acknowledgment timeout (60 seconds)
- Triggers escalation to backup contacts if no acknowledgment received
- Records response times for performance metrics
- Provides webhook endpoint for acknowledgment callbacks

### 10. Dashboard & Monitoring

**Responsibility**: Provides operator interface for monitoring, review, and system management.

**Interface**:
```
Dashboard:
  - getActiveEvents(): List<EmergencyEvent>
  - getEventHistory(filters: EventFilters, pagination: Pagination): PagedResult<EmergencyEvent>
  - getSystemHealth(): HealthStatus
  - getDetectorMetrics(timeRange: TimeRange): DetectorMetrics
  - reviewEvent(eventId: string, operatorId: string): EventReviewSession

HealthStatus:
  - overallStatus: enum(healthy, degraded, critical)
  - componentStatuses: Map<ComponentName, ComponentHealth>
  - activeAlerts: List<SystemAlert>

DetectorMetrics:
  - detectionCounts: Map<IncidentType, int>
  - falsePositiveRates: Map<IncidentType, float>
  - averageResponseTimes: Map<IncidentType, duration>
  - accuracyMetrics: Map<IncidentType, float>
```

**Behavior**:
- Real-time display of active events and system status
- Historical event search and filtering
- Operator review interface for validating detections
- System health monitoring with alerting
- Performance metrics and reporting

## Data Models

### Core Entities

**EmergencyEvent**:
```
EmergencyEvent:
  - eventId: UUID (primary key)
  - incidentType: enum(violence, collapse, fire)
  - severity: enum(low, medium, high, critical)
  - feedId: string (foreign key to Camera)
  - location: Location (embedded)
  - detectionTimestamp: timestamp
  - confidence: float
  - status: enum(pending, acknowledged, resolved, false_positive)
  - videoClipId: UUID (foreign key to VideoClip)
  - assignedAuthorityId: UUID (foreign key to Authority)
  - createdAt: timestamp
  - updatedAt: timestamp
  - acknowledgedAt: optional<timestamp>
  - acknowledgedBy: optional<string>
  - resolvedAt: optional<timestamp>
  - falsePositiveMarkedBy: optional<string>
  - falsePositiveReason: optional<string>
```

**Camera**:
```
Camera:
  - feedId: UUID (primary key)
  - streamUrl: string
  - location: Location (embedded)
  - status: enum(active, inactive, error)
  - lastFrameTimestamp: timestamp
  - configuration: CameraConfig (embedded)
  - createdAt: timestamp
  - updatedAt: timestamp

CameraConfig:
  - violenceDetectionEnabled: boolean
  - violenceThreshold: float
  - collapseDetectionEnabled: boolean
  - collapseThreshold: float
  - fireDetectionEnabled: boolean
  - fireThreshold: float
  - sensitivityLevel: enum(low, medium, high)
```

**Authority**:
```
Authority:
  - authorityId: UUID (primary key)
  - type: enum(police, hospital, fire_brigade)
  - name: string
  - location: Location (embedded)
  - serviceArea: GeoJSON (polygon)
  - primaryContact: ContactChannel (embedded)
  - backupContacts: List<ContactChannel> (embedded)
  - averageResponseTime: duration
  - active: boolean
  - createdAt: timestamp
  - updatedAt: timestamp
```

**Alert**:
```
Alert:
  - alertId: UUID (primary key)
  - eventId: UUID (foreign key to EmergencyEvent)
  - authorityId: UUID (foreign key to Authority)
  - sentAt: timestamp
  - acknowledgedAt: optional<timestamp>
  - acknowledgedBy: optional<string>
  - escalated: boolean
  - escalatedAt: optional<timestamp>
  - channelResults: List<ChannelSendResult> (embedded)
  - status: enum(sent, acknowledged, escalated, failed)
```

**VideoClip**:
```
VideoClip:
  - clipId: UUID (primary key)
  - eventId: UUID (foreign key to EmergencyEvent)
  - feedId: UUID (foreign key to Camera)
  - startTime: timestamp
  - endTime: timestamp
  - duration: duration
  - storageUrl: string
  - encrypted: boolean
  - encryptionKey: string (encrypted)
  - sizeBytes: int
  - retentionExpiresAt: timestamp
  - accessLog: List<AccessRecord> (embedded)
  - createdAt: timestamp

AccessRecord:
  - accessedBy: string
  - accessedAt: timestamp
  - purpose: string
```

### Database Schema Considerations

- **Primary Database**: PostgreSQL for transactional data (events, alerts, authorities, cameras)
- **Video Storage**: Object storage (S3-compatible) for video clips
- **Cache Layer**: Redis for authority location cache and feed status
- **Time-Series Database**: InfluxDB or Prometheus for metrics and monitoring data

**Indexes**:
- EmergencyEvent: (status, detectionTimestamp), (feedId, detectionTimestamp)
- Alert: (eventId), (status, sentAt)
- Camera: (status), (location) for geospatial queries
- Authority: (type, location) for geospatial queries


## Correctness Properties

*A property is a characteristic or behavior that should hold true across all valid executions of a system—essentially, a formal statement about what the system should do. Properties serve as the bridge between human-readable specifications and machine-verifiable correctness guarantees.*

### Video Processing Properties

**Property 1: Continuous frame processing**
*For any* sequence of video frames from a CCTV feed, the Video_Analyzer should process all frames without dropping any, maintaining continuous operation.
**Validates: Requirements 1.1**

**Property 2: Feed startup responsiveness**
*For any* newly available CCTV feed, the Video_Analyzer should begin processing frames within 5 seconds of feed registration.
**Validates: Requirements 1.2**

**Property 3: Frame processing latency bound**
*For any* video frame, the Video_Analyzer should complete processing in under 2 seconds.
**Validates: Requirements 1.3**

**Property 4: Disconnection handling**
*For any* CCTV feed disconnection event, the system should create a log entry and initiate reconnection attempts.
**Validates: Requirements 1.4**

**Property 5: Concurrent feed processing**
*For any* set of N CCTV feeds (where N ≥ 1), the Video_Analyzer should process all feeds concurrently without errors or frame drops.
**Validates: Requirements 1.5**

### Detection Properties

**Property 6: Violence event creation**
*For any* video sequence containing violence (from test dataset), the Video_Analyzer should detect it and create an EmergencyEvent with incidentType=violence.
**Validates: Requirements 2.1**

**Property 7: Violence severity classification**
*For any* detected violence event, the severity field should be one of {low, medium, high, critical}.
**Validates: Requirements 2.2**

**Property 8: Collapse event creation**
*For any* video sequence containing a person collapsing (from test dataset), the Video_Analyzer should detect it and create an EmergencyEvent with incidentType=collapse.
**Validates: Requirements 3.1**

**Property 9: Intentional sitting/lying rejection**
*For any* video sequence showing intentional sitting or lying down (from test dataset), the Video_Analyzer should not create an emergency event.
**Validates: Requirements 3.2**

**Property 10: Motionless escalation**
*For any* collapse event where the person remains motionless for more than 30 seconds, the system should escalate the event severity to critical.
**Validates: Requirements 3.5**

**Property 11: Fire event creation**
*For any* video sequence containing fire or smoke (from test dataset), the Video_Analyzer should detect it and create an EmergencyEvent with incidentType=fire.
**Validates: Requirements 4.1**

**Property 12: Fire subtype detection**
*For any* video sequence containing visible flames OR smoke patterns, the Video_Analyzer should create a fire event (both subtypes detected).
**Validates: Requirements 4.2**

### Evidence Capture Properties

**Property 13: Evidence capture timing**
*For any* detected emergency event (violence or collapse), the captured video clip should start 10 seconds before the detection timestamp and end 10 seconds after. For fire events, the clip should start 10 seconds before and continue until event resolution.
**Validates: Requirements 2.3, 3.3, 4.3**

### Authority Routing Properties

**Property 14: Incident-to-authority type mapping**
*For any* emergency event, the Authority_Locator should return an authority matching the incident type: violence→police, collapse→hospital, fire→fire_brigade.
**Validates: Requirements 5.1, 5.2, 5.3**

**Property 15: Authority lookup performance**
*For any* location and incident type, the Authority_Locator should return the nearest authority within 3 seconds.
**Validates: Requirements 5.4**

### Location Properties

**Property 16: Location determination**
*For any* emergency event, the system should determine and attach a Location object to the event.
**Validates: Requirements 6.1**

**Property 17: Location data completeness**
*For any* location object, it should contain buildingName, cameraId, and either (address OR mapReference). If GPS coordinates are available in the camera metadata, they should be included.
**Validates: Requirements 6.2, 6.3, 6.4**

**Property 18: Location validation before alert**
*For any* alert generation attempt, if the location data is incomplete (missing required fields), the alert generation should fail with a validation error.
**Validates: Requirements 6.5**

### Alert Generation and Dispatch Properties

**Property 19: Alert generation timing**
*For any* emergency event with an identified authority, the Alert_Dispatcher should generate an alert within 5 seconds.
**Validates: Requirements 7.1**

**Property 20: Alert data completeness**
*For any* generated alert, it should include incidentType, location, timestamp, severity, and videoEvidenceUrl fields.
**Validates: Requirements 7.2**

**Property 21: Multi-channel dispatch**
*For any* alert with configured contact channels, the system should attempt to send the alert through all channels (SMS, email, webhook).
**Validates: Requirements 7.3**

**Property 22: Alert transmission logging**
*For any* sent alert, the system should create a log entry with transmission details and activate acknowledgment tracking.
**Validates: Requirements 7.4**

**Property 23: Unacknowledged alert escalation**
*For any* alert that remains unacknowledged for more than 60 seconds, the system should escalate to backup authority contacts.
**Validates: Requirements 7.5**

### Alert Tracking Properties

**Property 24: Alert lifecycle event tracking**
*For any* alert delivery or acknowledgment event, the system should record a timestamp and (for acknowledgments) the responder identity.
**Validates: Requirements 8.1, 8.2**

**Property 25: Event status validity**
*For any* emergency event at any point in time, its status should be one of {pending, acknowledged, resolved, false_positive}.
**Validates: Requirements 8.3**

**Property 26: Dashboard event query completeness**
*For any* dashboard query with filters, the returned results should include all events matching the filter criteria and no events that don't match.
**Validates: Requirements 8.4**

**Property 27: Report generation**
*For any* time range, the system should generate a report containing response time metrics and incident outcome statistics for all events in that range.
**Validates: Requirements 8.5**

### False Positive Handling Properties

**Property 28: False positive marking capability**
*For any* emergency event, an operator should be able to mark it as false_positive, and the operation should succeed.
**Validates: Requirements 9.1**

**Property 29: False positive feedback persistence**
*For any* event marked as false positive with feedback, the feedback should be stored and retrievable from the database.
**Validates: Requirements 9.2**

**Property 30: Manual review queueing**
*For any* detection when manual review mode is enabled, the event should be added to the review queue instead of triggering immediate alert dispatch.
**Validates: Requirements 9.4**

**Property 31: False positive rate tracking**
*For any* false positive marking, the system should update the false positive rate metrics for the corresponding camera and incident type.
**Validates: Requirements 9.5**

### Reliability and Failover Properties

**Property 32: Feed redistribution on analyzer failure**
*For any* Video_Analyzer instance failure, all feeds assigned to that instance should be redistributed to healthy instances within 30 seconds.
**Validates: Requirements 10.1**

**Property 33: Authority database fallback**
*For any* authority lookup when the primary database is unavailable, the system should successfully return results from the cached backup.
**Validates: Requirements 10.3**

**Property 34: Component failure alerting**
*For any* component health check failure, the system should generate an administrator alert within 10 seconds.
**Validates: Requirements 10.4**

### Privacy and Security Properties

**Property 35: Video data encryption**
*For any* stored video clip, the data should be encrypted (encrypted flag = true and encryption key present).
**Validates: Requirements 11.1**

**Property 36: Event video retention**
*For any* video clip associated with an emergency event, the retention expiration timestamp should be set to 90 days from creation.
**Validates: Requirements 11.2**

**Property 37: Non-event video deletion**
*For any* video clip not associated with an emergency event, the retention expiration timestamp should be set to 24 hours from creation.
**Validates: Requirements 11.3**

**Property 38: Video access authorization**
*For any* video access attempt by a user without proper role permissions, the access should be denied.
**Validates: Requirements 11.4**

**Property 39: Video access audit logging**
*For any* successful video access, an AccessRecord should be created with userId and timestamp.
**Validates: Requirements 11.5**

### Configuration Properties

**Property 40: Configuration round-trip**
*For any* camera configuration update (sensitivity, thresholds, enabled detection types), retrieving the configuration should return the updated values.
**Validates: Requirements 12.1, 12.2, 12.3**

**Property 41: Configuration application timing**
*For any* configuration change, the new configuration should be applied and take effect within 10 seconds.
**Validates: Requirements 12.4**

**Property 42: Configuration validation**
*For any* invalid configuration value (e.g., threshold > 1.0, negative sensitivity), the system should reject the configuration change with a validation error.
**Validates: Requirements 12.5**

## Error Handling

### Detection Errors

**Model Inference Failures**:
- If ML model inference fails for a frame, log the error with frame metadata
- Continue processing subsequent frames (don't halt the entire feed)
- Alert administrators if inference failure rate exceeds 5% over a 5-minute window
- Fallback: Skip the problematic frame and continue

**Low Confidence Detections**:
- If detection confidence is below configured threshold, do not create an event
- Log low-confidence detections for model improvement analysis
- Provide operator dashboard to review borderline cases

### Feed Connection Errors

**Stream Unavailable**:
- Implement exponential backoff reconnection: 1s, 2s, 4s, 8s, 16s, 32s, 60s (max)
- After 10 consecutive failures, mark feed as "error" status and alert administrators
- Continue reconnection attempts indefinitely until manual intervention

**Frame Corruption**:
- If frame decoding fails, log the error and skip to next frame
- If corruption rate exceeds 10% over 1 minute, mark feed as degraded
- Alert administrators for persistent corruption issues

### Authority Routing Errors

**No Authority Found**:
- If no authority is found within service area, expand search radius by 50%
- If still no authority found, use regional default authority
- Log all cases where default authority is used for review

**Authority Database Unavailable**:
- Use cached authority data (updated every 5 minutes)
- If cache is also unavailable, use hardcoded emergency contacts
- Alert administrators immediately about database unavailability

### Alert Dispatch Errors

**Channel Send Failure**:
- Retry failed channel with exponential backoff: 2s, 4s, 8s (3 attempts max)
- If all channels fail, log critical error and alert system administrators
- Continue with escalation process even if primary alert fails

**Acknowledgment Timeout**:
- After 60 seconds without acknowledgment, escalate to backup contacts
- After 120 seconds without acknowledgment from backup, alert system administrators
- Log all timeout events for response time analysis

### Data Storage Errors

**Video Storage Failure**:
- If video clip upload fails, retry up to 3 times with 5-second delays
- If all retries fail, store clip metadata without video URL
- Alert administrators about storage failures
- Queue failed uploads for batch retry

**Database Write Failure**:
- Retry database writes up to 3 times with exponential backoff
- If event creation fails, queue event in memory buffer for retry
- If buffer exceeds 1000 events, write to local disk as backup
- Alert administrators about persistent database issues

### Privacy and Security Errors

**Encryption Failure**:
- If video encryption fails, do not store the video clip
- Log encryption failure with clip metadata
- Alert security team immediately
- Retry encryption once; if still fails, discard video

**Access Control Violation**:
- Log all unauthorized access attempts with user identity and timestamp
- Block user after 3 consecutive unauthorized attempts
- Alert security team about access violations
- Maintain audit trail for compliance

## Testing Strategy

### Dual Testing Approach

The Emergency Detection System requires both unit testing and property-based testing for comprehensive validation:

**Unit Tests**: Focus on specific examples, edge cases, and integration points
- Specific video sequences with known outcomes
- Boundary conditions (exactly 30 seconds motionless, exactly 60 seconds for escalation)
- Error conditions (malformed data, network failures)
- Integration between components (event creation → authority lookup → alert dispatch)

**Property-Based Tests**: Verify universal properties across all inputs
- Detection properties with randomized test video sequences
- Timing properties with varied system loads
- Data completeness properties with generated events
- Configuration properties with random valid/invalid values

Together, these approaches provide comprehensive coverage: unit tests catch concrete bugs in specific scenarios, while property tests verify general correctness across the input space.

### Property-Based Testing Configuration

**Framework Selection**:
- Python: Use Hypothesis library for property-based testing
- TypeScript/JavaScript: Use fast-check library
- Each property test should run minimum 100 iterations due to randomization

**Test Tagging**:
Each property-based test must include a comment tag referencing the design document property:
```python
# Feature: emergency-detection-system, Property 1: Continuous frame processing
def test_continuous_frame_processing():
    ...
```

**Property Test Implementation**:
- Each correctness property listed above must be implemented as a single property-based test
- Tests should generate random valid inputs within the domain
- Tests should verify the property holds for all generated inputs
- Tests should fail fast with clear counterexamples when properties are violated

### Test Data Requirements

**Video Test Dataset**:
- Curated dataset of labeled video sequences for each incident type
- Violence: 100+ sequences with varied scenarios (fighting, assault, aggressive behavior)
- Collapse: 100+ sequences (falls, fainting, medical emergencies)
- Fire: 100+ sequences (flames, smoke, various fire sizes)
- Negative examples: 200+ sequences (normal activity, intentional sitting/lying, steam/fog)

**Synthetic Data Generation**:
- Generate random EmergencyEvent objects with valid field values
- Generate random Location objects with varied completeness
- Generate random Camera configurations with valid/invalid parameters
- Generate random Authority databases with geographic distribution

### Integration Testing

**End-to-End Scenarios**:
1. Feed registration → frame processing → detection → event creation → authority lookup → alert dispatch → acknowledgment
2. Detection → manual review → false positive marking → feedback storage
3. Analyzer failure → feed redistribution → continued processing
4. Database failure → cache fallback → continued operation

**Performance Testing**:
- Load testing with 50+ concurrent CCTV feeds
- Latency testing for detection and alert generation timing requirements
- Stress testing for failover and recovery scenarios

### Continuous Validation

**Model Accuracy Monitoring**:
- Maintain separate validation dataset (not used in training)
- Run weekly accuracy validation against labeled test set
- Track detection accuracy, false positive rates, and false negative rates
- Alert ML team if accuracy drops below requirements (85% violence, 80% collapse, 90% fire)

**False Positive Analysis**:
- Review all operator-marked false positives weekly
- Analyze patterns in false positives (time of day, camera location, environmental conditions)
- Use feedback to retrain and improve detection models
- Track false positive rate trends over time

### Compliance Testing

**Privacy Validation**:
- Verify encryption for all stored video clips
- Verify retention policies are enforced (90 days for events, 24 hours for non-events)
- Verify access control prevents unauthorized video access
- Verify audit logs capture all video access

**Security Testing**:
- Penetration testing for API endpoints
- Access control testing for all user roles
- Encryption validation for data in transit and at rest
- Audit log integrity verification
