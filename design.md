# Design Document: Food Donation Matching

## Overview

The Food Donation Matching platform is a real-time, AI-powered system that connects food donors with receivers through volunteer-based logistics. The system operates similarly to ride-sharing platforms, with sub-30-second response times for critical operations and intelligent matching algorithms that optimize for freshness, proximity, and social impact.

The architecture follows a microservices pattern with event-driven communication, enabling independent scaling of matching, routing, and notification services. The system uses geospatial indexing for proximity queries, machine learning models for spoilage prediction and demand forecasting, and a mobile-first API design with offline-first capabilities.

## Architecture

### High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        Client Layer                              │
│  (Mobile Web, Progressive Web App, SMS Gateway)                  │
└────────────────┬────────────────────────────────────────────────┘
                 │
┌────────────────▼────────────────────────────────────────────────┐
│                     API Gateway Layer                            │
│  (Authentication, Rate Limiting, Request Routing)                │
└────────────────┬────────────────────────────────────────────────┘
                 │
┌────────────────▼────────────────────────────────────────────────┐
│                   Core Services Layer                            │
│                                                                   │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐          │
│  │   Posting    │  │   Matching   │  │   Routing    │          │
│  │   Service    │  │   Service    │  │   Service    │          │
│  └──────────────┘  └──────────────┘  └──────────────┘          │
│                                                                   │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐          │
│  │ Notification │  │   Tracking   │  │ Gamification │          │
│  │   Service    │  │   Service    │  │   Service    │          │
│  └──────────────┘  └──────────────┘  └──────────────┘          │
└────────────────┬────────────────────────────────────────────────┘
                 │
┌────────────────▼────────────────────────────────────────────────┐
│                    AI/ML Services Layer                          │
│                                                                   │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐          │
│  │  Spoilage    │  │   Demand     │  │    Route     │          │
│  │  Predictor   │  │  Forecaster  │  │  Optimizer   │          │
│  └──────────────┘  └──────────────┘  └──────────────┘          │
└────────────────┬────────────────────────────────────────────────┘
                 │
┌────────────────▼────────────────────────────────────────────────┐
│                      Data Layer                                  │
│                                                                   │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐          │
│  │  PostgreSQL  │  │    Redis     │  │   MongoDB    │          │
│  │  (Relational)│  │   (Cache)    │  │  (Geospatial)│          │
│  └──────────────┘  └──────────────┘  └──────────────┘          │
└─────────────────────────────────────────────────────────────────┘
```

### Technology Stack

- **API Gateway**: Kong or AWS API Gateway with JWT authentication
- **Real-Time Communication**: WebSocket (Socket.io) for live updates, Server-Sent Events for notifications
- **Core Services**: Node.js/TypeScript or Python (FastAPI) microservices
- **Message Queue**: Apache Kafka or RabbitMQ for event-driven communication
- **Geospatial Database**: MongoDB with 2dsphere indexes or PostGIS
- **Cache Layer**: Redis for session management, real-time volunteer locations, and hot data
- **ML Framework**: Python with scikit-learn, TensorFlow Lite for edge deployment
- **SMS Gateway**: Twilio or AWS SNS for offline notifications
- **Mobile Client**: Progressive Web App (PWA) with service workers for offline support

### Design Principles

1. **Real-Time First**: All critical operations (notifications, tracking, matching) complete within 30 seconds
2. **Offline Resilient**: Volunteers can operate with cached data and sync when connectivity returns
3. **AI-Augmented**: Machine learning enhances but never blocks core operations (fallback to rule-based logic)
4. **Privacy-Preserving**: Location data encrypted at rest, shared only with authorized parties
5. **Scalable by City**: Data partitioned by city for independent scaling and regulatory compliance

## Components and Interfaces

### 1. Posting Service

**Responsibility**: Manages food posting lifecycle (create, update, cancel, expire)

**Key Operations**:
- `createPosting(donorId, foodDetails, pickupDeadline, storageConditions)`: Creates a new food posting
- `updatePosting(postingId, updates)`: Updates posting details with history tracking
- `cancelPosting(postingId, reason)`: Cancels posting and triggers notifications
- `expirePosting(postingId)`: Marks posting as expired when spoilage window exceeded

**Data Model**:
```typescript
interface FoodPosting {
  id: string;
  donorId: string;
  foodType: string;
  quantity: number; // in kg
  quantityUnit: 'kg' | 'servings';
  freshnessIndicator: 'fresh' | 'cooked_today' | 'cooked_yesterday';
  preparationTime: Date;
  storageConditions: 'refrigerated' | 'frozen' | 'room_temperature';
  pickupDeadline: Date;
  spoilageWindow: Date; // AI-calculated
  location: GeoPoint;
  status: 'available' | 'claimed' | 'in_transit' | 'delivered' | 'expired' | 'cancelled';
  claimedBy?: string; // receiverId
  assignedVolunteer?: string;
  createdAt: Date;
  updatedAt: Date;
  history: PostingHistoryEntry[];
}

interface GeoPoint {
  type: 'Point';
  coordinates: [number, number]; // [longitude, latitude]
}
```

**Events Emitted**:
- `posting.created`: Triggers matching and notification services
- `posting.updated`: Updates interested parties
- `posting.cancelled`: Triggers re-matching if claimed
- `posting.expired`: Updates analytics

### 2. Matching Service

**Responsibility**: Intelligently matches food postings with receivers based on proximity, capacity, dietary restrictions, and spoilage urgency

**Key Operations**:
- `findEligibleReceivers(postingId, radius)`: Returns ranked list of receivers
- `calculateMatchScore(posting, receiver)`: Computes match quality score
- `notifyReceivers(postingId, receivers)`: Triggers notification service
- `handleEmergencyRematch(postingId)`: Expands radius and broadcasts urgently

**Matching Algorithm**:

```
MatchScore = (ProximityScore * 0.3) + 
             (SpoilageUrgencyScore * 0.4) + 
             (ReceiverReliabilityScore * 0.2) + 
             (DietaryCompatibilityScore * 0.1)

Where:
- ProximityScore = 1 - (distance / maxRadius)
- SpoilageUrgencyScore = 1 - (remainingTime / totalSpoilageWindow)
- ReceiverReliabilityScore = historicalAcceptanceRate
- DietaryCompatibilityScore = 1 if compatible, 0 otherwise
```

**Geospatial Query**:
```javascript
// MongoDB geospatial query for receivers within radius
db.receivers.find({
  location: {
    $near: {
      $geometry: { type: "Point", coordinates: [donorLng, donorLat] },
      $maxDistance: radiusInMeters
    }
  },
  status: "active",
  cityId: posting.cityId
})
```

### 3. Routing Service

**Responsibility**: Optimizes volunteer routes from current location → donor → receiver

**Key Operations**:
- `findAvailableVolunteers(donorLocation, radius)`: Returns volunteers within range
- `assignVolunteer(postingId, volunteerId)`: Assigns volunteer and generates route
- `optimizeRoute(volunteerLocation, donorLocation, receiverLocation)`: Calculates optimal path
- `updateVolunteerLocation(volunteerId, location)`: Updates real-time position

**Route Optimization**:
- Uses Dijkstra's algorithm or A* for pathfinding with real-time traffic data
- Integrates with Google Maps Directions API or Mapbox for production routing
- Fallback to straight-line distance calculation when API unavailable
- Considers pickup deadline as hard constraint

**Data Model**:
```typescript
interface Volunteer {
  id: string;
  name: string;
  phone: string;
  currentLocation: GeoPoint;
  status: 'available' | 'assigned' | 'in_transit' | 'offline';
  reliabilityScore: number; // 0-100
  socialCredits: number;
  activeAssignment?: string; // postingId
  cityId: string;
  lastLocationUpdate: Date;
}

interface Route {
  volunteerId: string;
  postingId: string;
  waypoints: [GeoPoint, GeoPoint, GeoPoint]; // [volunteer, donor, receiver]
  estimatedDistance: number; // in km
  estimatedDuration: number; // in minutes
  actualDistance?: number;
  actualDuration?: number;
  createdAt: Date;
}
```

### 4. Notification Service

**Responsibility**: Delivers real-time notifications via WebSocket, push notifications, and SMS fallback

**Key Operations**:
- `notifyReceivers(postingId, receiverIds, priority)`: Sends posting notifications
- `notifyStatusUpdate(postingId, status, recipients)`: Broadcasts status changes
- `sendSMSFallback(userId, message)`: Sends SMS when network unavailable
- `broadcastEmergency(postingId, message)`: Emergency broadcast for late cancellations

**Notification Channels**:
1. **WebSocket**: Primary channel for real-time updates (sub-second latency)
2. **Push Notifications**: For mobile app users when app is backgrounded
3. **SMS**: Fallback for poor connectivity or critical updates

**Message Format**:
```typescript
interface Notification {
  id: string;
  type: 'posting_available' | 'posting_claimed' | 'status_update' | 'emergency';
  priority: 'normal' | 'high' | 'urgent';
  recipientId: string;
  postingId: string;
  message: string;
  data: Record<string, any>;
  channels: ('websocket' | 'push' | 'sms')[];
  sentAt: Date;
  deliveredAt?: Date;
}
```

### 5. Tracking Service

**Responsibility**: Manages real-time location tracking and status updates for deliveries

**Key Operations**:
- `updateVolunteerLocation(volunteerId, location)`: Updates volunteer position every 30s
- `updateDeliveryStatus(postingId, status, metadata)`: Updates posting status
- `getDeliveryTracking(postingId)`: Returns real-time tracking data
- `calculateETA(volunteerId, destinationLocation)`: Estimates arrival time

**Status State Machine**:
```
available → claimed → assigned → en_route_pickup → picking_up → 
en_route_delivery → delivering → delivered

Alternative paths:
- any_state → cancelled (by donor/receiver)
- any_state → expired (spoilage window exceeded)
```

### 6. Gamification Service

**Responsibility**: Manages volunteer social credits, achievements, and leaderboards

**Key Operations**:
- `awardCredits(volunteerId, deliveryId, baseCredits, bonusCredits)`: Awards credits for completed delivery
- `updateReliabilityScore(volunteerId, action, delta)`: Adjusts reliability score
- `checkAchievements(volunteerId)`: Checks and unlocks achievement badges
- `getLeaderboard(cityId, period)`: Returns top volunteers for time period

**Credit Calculation**:
```
TotalCredits = BaseCredits + DistanceBonus + SpeedBonus

Where:
- BaseCredits = 10
- DistanceBonus = distance_km * 1
- SpeedBonus = 5 if delivered before 80% of spoilage window, else 0
```

**Achievement Badges**:
- First Delivery: Complete 1 delivery
- Reliable Helper: Maintain reliability score > 90 for 30 days
- Speed Demon: Complete 10 deliveries before 80% spoilage window
- Community Champion: Complete 100 deliveries
- City Hero: Top 10 on monthly leaderboard

### 7. Analytics Service

**Responsibility**: Aggregates platform metrics and generates insights for admins and donors

**Key Operations**:
- `calculateImpactMetrics(cityId, timeRange)`: Returns food saved, meals delivered, carbon footprint
- `generateDonorReport(donorId, timeRange)`: Donor-specific waste patterns
- `generateVolunteerReport(volunteerId)`: Volunteer performance metrics
- `exportAnalytics(filters, format)`: Exports data for external analysis

**Metrics Tracked**:

```typescript
interface ImpactMetrics {
  totalFoodSaved: number; // kg
  totalMealsDelivered: number; // calculated as foodSaved * 0.4
  carbonFootprintReduced: number; // kg CO2e
  totalDeliveries: number;
  averageDeliveryTime: number; // minutes
  successRate: number; // percentage
  topDonors: DonorSummary[];
  topVolunteers: VolunteerSummary[];
}

// Carbon footprint calculation
// Food waste emissions: ~2.5 kg CO2e per kg food wasted (IPCC estimates)
// Transport emissions: ~0.12 kg CO2e per km (small vehicle)
CarbonReduced = (foodSaved_kg * 2.5) - (totalDistance_km * 0.12)
```

## Data Models

### Core Entities

**User** (Base entity for Donor, Receiver, Volunteer):
```typescript
interface User {
  id: string;
  type: 'donor' | 'receiver' | 'volunteer';
  name: string;
  email: string;
  phone: string;
  location: GeoPoint;
  address: string;
  cityId: string;
  status: 'active' | 'inactive' | 'suspended';
  disclaimerAcceptedAt?: Date;
  disclaimerVersion: string;
  createdAt: Date;
  updatedAt: Date;
}
```

**Donor** (extends User):
```typescript
interface Donor extends User {
  type: 'donor';
  organizationType: 'hotel' | 'hostel' | 'canteen' | 'restaurant' | 'other';
  totalPostings: number;
  totalFoodDonated: number; // kg
  wasteReductionPercentage: number;
}
```

**Receiver** (extends User):
```typescript
interface Receiver extends User {
  type: 'receiver';
  organizationType: 'ngo' | 'shelter' | 'community_kitchen' | 'other';
  capacity: number; // people served daily
  dietaryRestrictions: string[]; // ['vegetarian', 'halal', 'kosher', etc.]
  acceptanceRate: number; // historical claim-to-completion rate
  totalReceived: number; // kg
}
```

**Delivery**:
```typescript
interface Delivery {
  id: string;
  postingId: string;
  donorId: string;
  receiverId: string;
  volunteerId: string;
  route: Route;
  status: DeliveryStatus;
  pickupTime?: Date;
  deliveryTime?: Date;
  creditsAwarded: number;
  reliabilityImpact: number;
  createdAt: Date;
  completedAt?: Date;
}
```

### Database Schema Design

**Partitioning Strategy**:
- Partition by `cityId` for horizontal scaling
- Each city operates as independent shard
- Enables city-specific regulations and configurations

**Indexes**:
```sql
-- PostgreSQL indexes for relational data
CREATE INDEX idx_postings_status_city ON postings(status, city_id);
CREATE INDEX idx_postings_spoilage ON postings(spoilage_window) WHERE status = 'available';
CREATE INDEX idx_users_city_type ON users(city_id, type);

-- MongoDB geospatial indexes
db.postings.createIndex({ location: "2dsphere", cityId: 1 });
db.volunteers.createIndex({ currentLocation: "2dsphere", status: 1, cityId: 1 });
db.receivers.createIndex({ location: "2dsphere", cityId: 1 });
```

**Caching Strategy**:
- Redis cache for active volunteer locations (TTL: 60 seconds)
- Redis cache for available postings by city (TTL: 30 seconds)
- Redis cache for user sessions (TTL: 24 hours)

## AI/ML Components

### 1. Spoilage Prediction Model

**Purpose**: Predict food spoilage window based on food type, preparation time, storage conditions, and environmental factors

**Model Type**: Gradient Boosting (XGBoost or LightGBM)

**Features**:
- Food type (categorical: vegetables, cooked_rice, curry, bread, etc.)
- Freshness indicator (categorical: fresh, cooked_today, cooked_yesterday)
- Storage conditions (categorical: refrigerated, frozen, room_temperature)
- Elapsed time since preparation (continuous: hours)
- Ambient temperature (continuous: °C) - from weather API
- Humidity (continuous: %) - from weather API
- Quantity (continuous: kg)

**Target Variable**: Spoilage window in hours

**Training Data**: Historical postings with actual spoilage observations (crowdsourced from receivers)

**Fallback Rules** (when ML unavailable):
```python
SPOILAGE_RULES = {
    ('fresh', 'refrigerated'): 48,  # hours
    ('fresh', 'room_temperature'): 12,
    ('cooked_today', 'refrigerated'): 24,
    ('cooked_today', 'room_temperature'): 6,
    ('cooked_yesterday', 'refrigerated'): 12,
    ('cooked_yesterday', 'room_temperature'): 3,
    ('frozen', 'frozen'): 720,  # 30 days
}
```

**Model Update Frequency**: Retrain weekly with new data

### 2. Demand Forecasting Model

**Purpose**: Predict receiver demand by food type, location, and time to enable proactive donor outreach

**Model Type**: Time Series Forecasting (LSTM or Prophet)

**Features**:
- Day of week (categorical)
- Hour of day (continuous)
- Food type (categorical)
- Receiver location cluster (categorical)
- Historical demand (time series)
- Seasonal indicators (holidays, festivals)
- Weather conditions

**Target Variable**: Expected demand (kg) by food type and location cluster

**Prediction Horizon**: 24 hours ahead

**Training Data**: Minimum 60 days of historical claims and deliveries

**Output**: Daily demand forecast by city, food type, and time window

### 3. Route Optimization

**Purpose**: Calculate optimal route for volunteer considering traffic, distance, and pickup deadline

**Approach**: 
- **Production**: Integrate with Google Maps Directions API or Mapbox
- **Fallback**: Haversine distance + average speed estimation

**Haversine Formula** (for geodesic distance):
```python
def haversine_distance(lat1, lon1, lat2, lon2):
    R = 6371  # Earth radius in km
    
    dlat = radians(lat2 - lat1)
    dlon = radians(lon2 - lon1)
    
    a = sin(dlat/2)**2 + cos(radians(lat1)) * cos(radians(lat2)) * sin(dlon/2)**2
    c = 2 * atan2(sqrt(a), sqrt(1-a))
    
    return R * c
```

**ETA Calculation**:
```python
# Average urban speed: 25 km/h (accounting for traffic, stops)
AVERAGE_SPEED_KMH = 25

def calculate_eta(distance_km):
    return (distance_km / AVERAGE_SPEED_KMH) * 60  # minutes
```

### 4. Matching Algorithm Enhancement

**Purpose**: Learn optimal matching weights from historical success rates

**Model Type**: Reinforcement Learning (optional) or Logistic Regression

**Features**:
- Distance between donor and receiver
- Spoilage urgency
- Receiver reliability score
- Dietary compatibility
- Historical acceptance patterns

**Target Variable**: Successful delivery (binary: 1 if delivered, 0 if cancelled/expired)

**Optimization Goal**: Maximize successful deliveries while minimizing food waste

**Initial Weights** (rule-based, can be tuned by ML):
- Proximity: 30%
- Spoilage Urgency: 40%
- Receiver Reliability: 20%
- Dietary Compatibility: 10%


## Correctness Properties

A property is a characteristic or behavior that should hold true across all valid executions of a system—essentially, a formal statement about what the system should do. Properties serve as the bridge between human-readable specifications and machine-verifiable correctness guarantees.

### Property 1: Food Posting Data Integrity

*For any* valid food posting data (quantity, food type, freshness, pickup deadline, storage conditions), when a posting is created, all provided fields should be stored correctly and retrievable.

**Validates: Requirements 1.1, 16.1, 16.2**

### Property 2: Future Deadline Validation

*For any* date value, when creating a food posting with that date as the pickup deadline, the posting should only be accepted if the date is in the future.

**Validates: Requirements 1.2**

### Property 3: Unique Posting Identifiers

*For any* set of food postings created, all posting IDs should be unique across the entire system.

**Validates: Requirements 1.3**

### Property 4: Posting History Preservation

*For any* food posting, when updates are made to the posting, the history should contain all previous versions with timestamps.

**Validates: Requirements 1.4**

### Property 5: Geospatial Proximity Filtering

*For any* food posting and configurable radius, when identifying eligible receivers, only receivers within the geodesic distance (calculated using Haversine formula) from the donor should be included.

**Validates: Requirements 2.1, 2.2, 11.2**

### Property 6: Dietary Restriction Filtering

*For any* receiver with dietary restrictions and any food posting, the receiver should only be notified if the food type is compatible with their dietary restrictions.

**Validates: Requirements 2.3**

### Property 7: Notification Ranking by Urgency

*For any* set of available food postings, when ranking notifications for a receiver, postings with higher spoilage urgency and closer proximity should rank higher.

**Validates: Requirements 2.4**

### Property 8: Claim State Transition

*For any* available food posting, when a receiver claims it, the posting status should change to "claimed" and be assigned to that receiver.

**Validates: Requirements 3.1**

### Property 9: Double-Claim Prevention

*For any* already-claimed food posting, when another receiver attempts to claim it, the claim should be rejected with an error.

**Validates: Requirements 3.2**

### Property 10: Cancellation State Reversal

*For any* claimed food posting, when the receiver cancels it, the posting should return to "available" status and be unassigned.

**Validates: Requirements 3.4**

### Property 11: Volunteer Proximity Selection

*For any* claimed food posting, when identifying available volunteers, only volunteers within the configurable radius of the donor location and with status "available" should be considered.

**Validates: Requirements 4.1**

### Property 12: Closest Volunteer Assignment

*For any* set of available volunteers, when assigning a volunteer to a posting, the volunteer with the shortest estimated travel time to the donor should be selected.

**Validates: Requirements 4.2**

### Property 13: Route Waypoint Correctness

*For any* volunteer assignment, when generating a route, the route should contain exactly three waypoints in order: volunteer current location, donor location, receiver location.

**Validates: Requirements 4.3**

### Property 14: Route Deadline Compliance

*For any* generated route, the estimated arrival time at the donor location should be before the pickup deadline.

**Validates: Requirements 4.4**

### Property 15: Spoilage Window Calculation

*For any* food posting with food type, freshness indicator, and storage conditions, when the posting is created, a spoilage window should be calculated and should be positive (future time).

**Validates: Requirements 5.1, 16.4**

### Property 16: Urgent Posting Prioritization

*For any* food posting with spoilage window less than 2 hours, the posting should have higher priority in matching scores than postings with longer spoilage windows.

**Validates: Requirements 5.2**

### Property 17: Dynamic Radius Expansion

*For any* unclaimed food posting, when it reaches 80% of its spoilage window, the notification radius should be expanded by 50% from the original radius.

**Validates: Requirements 5.3**

### Property 18: Automatic Expiration

*For any* food posting, when the current time exceeds the spoilage window, the posting status should automatically change to "expired".

**Validates: Requirements 5.4**

### Property 19: Multi-Factor Match Scoring

*For any* donor-receiver pair, when calculating match score, the score should incorporate proximity, receiver capacity, dietary compatibility, and spoilage urgency with appropriate weights.

**Validates: Requirements 6.1**

### Property 20: Reliability-Based Tie Breaking

*For any* two receivers equidistant from a donor, the receiver with the higher historical acceptance rate should receive a higher match score.

**Validates: Requirements 6.2**

### Property 21: Adaptive Notification Filtering

*For any* receiver who has rejected a specific food type multiple times (>3), the frequency of notifications for that food type should be reduced.

**Validates: Requirements 6.3**

### Property 22: Urgency Weight Adjustment

*For any* perishable food posting (spoilage window < 6 hours), when calculating match scores, the spoilage urgency weight should be higher than the proximity weight.

**Validates: Requirements 6.4**

### Property 23: Delivery Status State Machine

*For any* delivery, status transitions should follow the valid state machine: available → claimed → assigned → en_route_pickup → picking_up → en_route_delivery → delivering → delivered, with no invalid transitions.

**Validates: Requirements 7.1, 7.3, 7.4, 7.5**

### Property 24: Social Credit Calculation

*For any* completed delivery, the total credits awarded should equal 10 (base) + distance_km * 1 (distance bonus) + 5 (speed bonus if delivered before 80% spoilage window).

**Validates: Requirements 8.1, 8.2, 8.4**

### Property 25: Achievement Milestone Detection

*For any* volunteer, when their total credits or delivery count reaches a milestone threshold, the corresponding achievement badge should be unlocked.

**Validates: Requirements 8.3**

### Property 26: Leaderboard Ranking Correctness

*For any* city and time period, the leaderboard should list volunteers in descending order by total credits earned in that period.

**Validates: Requirements 8.5**

### Property 27: Impact Metrics Aggregation

*For any* set of completed deliveries, the total food saved should equal the sum of all delivery quantities, and total meals should equal food saved * 0.4.

**Validates: Requirements 9.1, 9.2**

### Property 28: Carbon Footprint Calculation

*For any* completed delivery, the carbon footprint reduction should be calculated as (food_kg * 2.5) - (distance_km * 0.12).

**Validates: Requirements 9.3**

### Property 29: Volunteer Performance Metrics

*For any* volunteer, their performance metrics should accurately reflect their total deliveries, average delivery time, and reliability score based on their delivery history.

**Validates: Requirements 9.4**

### Property 30: Time-Series Aggregation

*For any* time period and city, analytics reports should correctly aggregate donations, claims, and deliveries within that time window.

**Validates: Requirements 9.5**

### Property 31: Donor Pattern Analysis

*For any* donor with historical postings, their analytics should identify patterns in food type, quantity, and timing based on their posting history.

**Validates: Requirements 10.1**

### Property 32: Waste Reduction Trend

*For any* donor with sufficient historical data, the waste reduction percentage should be calculated as the change in total donations over time.

**Validates: Requirements 10.4**

### Property 33: Geolocation Storage

*For any* user registration with valid coordinates, the geolocation should be stored and retrievable for that user.

**Validates: Requirements 11.1**

### Property 34: Location-Dependent Operation Blocking

*For any* location-dependent operation (matching, routing), when location services are unavailable, the operation should be blocked with an appropriate error.

**Validates: Requirements 11.4**

### Property 35: City Assignment by Location

*For any* user registration with coordinates, the user should be assigned to the city whose boundaries contain those coordinates.

**Validates: Requirements 12.1**

### Property 36: City-Scoped Matching

*For any* matching operation, only users (donors, receivers, volunteers) within the same city should be considered.

**Validates: Requirements 12.2**

### Property 37: City-Level Analytics Filtering

*For any* analytics query with city filter, the results should only include data from that specific city.

**Validates: Requirements 12.3**

### Property 38: City-Specific Configuration

*For any* city, matching radius and notification settings should be independently configurable and not affect other cities.

**Validates: Requirements 12.4**

### Property 39: Location Data Encryption

*For any* user location data stored in the database, the coordinates should be encrypted at rest.

**Validates: Requirements 14.1**

### Property 40: Location Access Control

*For any* volunteer's real-time location, only the assigned donor and receiver should have access to view that location data.

**Validates: Requirements 14.3**

### Property 41: User Data Deletion

*For any* user account deletion request, all associated personal data (profile, location history, postings, deliveries) should be removed from the system.

**Validates: Requirements 14.4**

### Property 42: Demand-Supply Threshold Notification

*For any* city and food type, when predicted demand exceeds recent supply by 30% or more, proactive notifications should be sent to nearby donors.

**Validates: Requirements 15.2**

### Property 43: Elapsed Time Calculation

*For any* food posting with preparation time, the elapsed time should be calculated as the difference between current time and preparation time.

**Validates: Requirements 16.3**

### Property 44: Emergency Re-Matching Trigger

*For any* claimed posting cancellation, when remaining time is less than 50% of the spoilage window, emergency re-matching should be automatically initiated with expanded radius (2x normal).

**Validates: Requirements 17.1, 17.2**

### Property 45: Urgent Priority Marking

*For any* emergency re-matching, the posting should be marked with "URGENT" priority in all subsequent notifications.

**Validates: Requirements 17.3**

### Property 46: Late Cancellation Penalty

*For any* late cancellation (< 50% spoilage window remaining), the receiver's reliability score should be decreased.

**Validates: Requirements 17.4**

### Property 47: Emergency Notification Content

*For any* emergency broadcast notification, the message should include the reason for urgency and remaining time until spoilage.

**Validates: Requirements 17.5**

### Property 48: Volunteer Reliability Score Initialization

*For any* volunteer accepting their first task, their reliability score should be initialized to 100.

**Validates: Requirements 18.1**

### Property 49: Reliability Score Updates

*For any* volunteer action (late arrival, cancellation, on-time completion), the reliability score should be adjusted by the correct amount (-5 for late, -10 for cancel, +2 for on-time) and capped at 100.

**Validates: Requirements 18.2, 18.3, 18.4**

### Property 50: High-Reliability Volunteer Prioritization

*For any* volunteer assignment, volunteers with reliability scores above 80 should be prioritized over those with lower scores.

**Validates: Requirements 18.5**

### Property 51: Low-Reliability Suspension

*For any* volunteer whose reliability score falls below 50, their status should be suspended and they should be unable to accept new assignments.

**Validates: Requirements 18.6**

### Property 52: Offline Data Caching

*For any* volunteer assignment, when network connectivity is lost, the assignment details should be cached locally and remain accessible.

**Validates: Requirements 19.2**

### Property 53: Online Synchronization

*For any* cached status updates, when network connectivity is restored, all cached updates should be synchronized with the server.

**Validates: Requirements 19.3, 19.5**

### Property 54: Network Quality Adaptive Fallback

*For any* notification, when network latency exceeds 3 seconds, the system should automatically switch to SMS mode for delivery.

**Validates: Requirements 19.6**

### Property 55: First-Time Disclaimer Requirement

*For any* donor creating their first posting, the system should require disclaimer acceptance before allowing the posting to be created.

**Validates: Requirements 20.1**

### Property 56: Claim Disclaimer Requirement

*For any* receiver claiming a posting, the system should require disclaimer acceptance before completing the claim.

**Validates: Requirements 20.2**

### Property 57: Disclaimer Acceptance Audit Trail

*For any* disclaimer acceptance, a timestamped record should be stored and retrievable for audit purposes.

**Validates: Requirements 20.3**

### Property 58: Annual Disclaimer Re-Acceptance

*For any* user whose last disclaimer acceptance was more than 365 days ago, the system should require re-acceptance before allowing critical operations.

**Validates: Requirements 20.6**


## Error Handling

### Error Categories

**1. Validation Errors** (HTTP 400)
- Invalid input data (negative quantities, past deadlines, invalid coordinates)
- Missing required fields (food type, preparation time, storage conditions)
- Format errors (invalid phone numbers, malformed addresses)

**Response Format**:
```json
{
  "error": "VALIDATION_ERROR",
  "message": "Pickup deadline must be in the future",
  "field": "pickupDeadline",
  "code": "INVALID_DEADLINE"
}
```

**2. Authorization Errors** (HTTP 403)
- Attempting to access another user's data
- Insufficient permissions for operation
- Suspended volunteer attempting to accept assignments

**Response Format**:
```json
{
  "error": "AUTHORIZATION_ERROR",
  "message": "Your account is temporarily suspended due to low reliability score",
  "code": "ACCOUNT_SUSPENDED",
  "suspensionEndsAt": "2024-01-15T00:00:00Z"
}
```

**3. Conflict Errors** (HTTP 409)
- Double-claiming a posting
- Updating an already-cancelled posting
- Volunteer accepting multiple assignments simultaneously

**Response Format**:
```json
{
  "error": "CONFLICT_ERROR",
  "message": "This posting has already been claimed by another receiver",
  "code": "ALREADY_CLAIMED",
  "claimedAt": "2024-01-08T10:30:00Z"
}
```

**4. Not Found Errors** (HTTP 404)
- Posting ID doesn't exist
- User not found
- City not configured

**Response Format**:
```json
{
  "error": "NOT_FOUND",
  "message": "Food posting not found",
  "code": "POSTING_NOT_FOUND",
  "postingId": "post_123456"
}
```

**5. Service Unavailable Errors** (HTTP 503)
- ML model unavailable (fallback to rule-based)
- External API failure (maps, SMS gateway)
- Database connection issues

**Response Format**:
```json
{
  "error": "SERVICE_UNAVAILABLE",
  "message": "Spoilage prediction service temporarily unavailable, using fallback rules",
  "code": "ML_SERVICE_DOWN",
  "fallbackUsed": true
}
```

### Graceful Degradation

**ML Service Failures**:
- Spoilage Prediction: Fall back to rule-based lookup table
- Demand Forecasting: Use historical averages
- Route Optimization: Use Haversine distance + average speed

**External API Failures**:
- Maps API: Fall back to Haversine distance calculation
- SMS Gateway: Queue messages for retry (exponential backoff)
- Weather API: Use cached data or default values

**Database Failures**:
- Read failures: Serve from cache if available, return 503 if not
- Write failures: Queue operations for retry, return 503 to client
- Connection pool exhaustion: Implement circuit breaker pattern

### Retry Strategies

**Idempotent Operations** (safe to retry):
- GET requests: Retry up to 3 times with exponential backoff
- Notification delivery: Retry up to 5 times over 10 minutes
- Analytics calculations: Retry indefinitely in background

**Non-Idempotent Operations** (use idempotency keys):
- Posting creation: Client provides idempotency key
- Claim operations: Use distributed lock (Redis)
- Credit awards: Transaction-based with deduplication

### Timeout Configuration

```typescript
const TIMEOUTS = {
  databaseQuery: 5000,        // 5 seconds
  externalAPI: 10000,         // 10 seconds
  mlPrediction: 3000,         // 3 seconds
  notificationDelivery: 2000, // 2 seconds
  websocketPing: 30000,       // 30 seconds
};
```

## Testing Strategy

### Dual Testing Approach

The system requires both unit testing and property-based testing for comprehensive coverage:

- **Unit tests**: Verify specific examples, edge cases, and error conditions
- **Property tests**: Verify universal properties across all inputs
- Both approaches are complementary and necessary

### Unit Testing

**Focus Areas**:
- Specific examples demonstrating correct behavior
- Edge cases (empty lists, boundary values, null handling)
- Error conditions (invalid inputs, service failures)
- Integration points between services
- Mock external dependencies (APIs, databases)

**Example Unit Tests**:
```typescript
describe('FoodPosting', () => {
  it('should reject posting with past deadline', () => {
    const pastDate = new Date('2020-01-01');
    expect(() => createPosting({ pickupDeadline: pastDate }))
      .toThrow('Pickup deadline must be in the future');
  });

  it('should handle empty volunteer list gracefully', () => {
    const result = assignVolunteer(postingId, []);
    expect(result.status).toBe('queued');
    expect(result.retryScheduled).toBe(true);
  });

  it('should calculate carbon footprint correctly for known values', () => {
    const delivery = { foodKg: 10, distanceKm: 5 };
    const carbon = calculateCarbonFootprint(delivery);
    expect(carbon).toBe((10 * 2.5) - (5 * 0.12));
  });
});
```

**Coverage Goals**:
- Line coverage: > 80%
- Branch coverage: > 75%
- Critical paths: 100%

### Property-Based Testing

**Configuration**:
- Minimum 100 iterations per property test
- Use fast-check (JavaScript/TypeScript) or Hypothesis (Python)
- Each test references its design document property

**Test Tag Format**:
```typescript
// Feature: food-donation-matching, Property 1: Food Posting Data Integrity
```

**Example Property Tests**:

```typescript
import fc from 'fast-check';

// Feature: food-donation-matching, Property 2: Future Deadline Validation
describe('Property: Future Deadline Validation', () => {
  it('should only accept future dates as pickup deadlines', () => {
    fc.assert(
      fc.property(
        fc.date(),
        (deadline) => {
          const now = new Date();
          const result = validatePickupDeadline(deadline);
          
          if (deadline > now) {
            expect(result.valid).toBe(true);
          } else {
            expect(result.valid).toBe(false);
            expect(result.error).toBe('INVALID_DEADLINE');
          }
        }
      ),
      { numRuns: 100 }
    );
  });
});

// Feature: food-donation-matching, Property 5: Geospatial Proximity Filtering
describe('Property: Geospatial Proximity Filtering', () => {
  it('should only include receivers within radius', () => {
    fc.assert(
      fc.property(
        fc.record({
          donorLat: fc.double({ min: -90, max: 90 }),
          donorLng: fc.double({ min: -180, max: 180 }),
          radius: fc.integer({ min: 1, max: 50 }), // km
          receivers: fc.array(fc.record({
            id: fc.uuid(),
            lat: fc.double({ min: -90, max: 90 }),
            lng: fc.double({ min: -180, max: 180 }),
          }))
        }),
        ({ donorLat, donorLng, radius, receivers }) => {
          const eligible = findEligibleReceivers(
            { lat: donorLat, lng: donorLng },
            radius,
            receivers
          );
          
          // All eligible receivers must be within radius
          eligible.forEach(receiver => {
            const distance = haversineDistance(
              donorLat, donorLng,
              receiver.lat, receiver.lng
            );
            expect(distance).toBeLessThanOrEqual(radius);
          });
          
          // All receivers within radius must be included
          receivers.forEach(receiver => {
            const distance = haversineDistance(
              donorLat, donorLng,
              receiver.lat, receiver.lng
            );
            if (distance <= radius) {
              expect(eligible).toContainEqual(receiver);
            }
          });
        }
      ),
      { numRuns: 100 }
    );
  });
});

// Feature: food-donation-matching, Property 24: Social Credit Calculation
describe('Property: Social Credit Calculation', () => {
  it('should calculate credits correctly for all deliveries', () => {
    fc.assert(
      fc.property(
        fc.record({
          distanceKm: fc.double({ min: 0.1, max: 100 }),
          deliveryTime: fc.integer({ min: 1, max: 300 }), // minutes
          spoilageWindow: fc.integer({ min: 60, max: 480 }), // minutes
        }),
        ({ distanceKm, deliveryTime, spoilageWindow }) => {
          const credits = calculateCredits(distanceKm, deliveryTime, spoilageWindow);
          
          const baseCredits = 10;
          const distanceBonus = Math.floor(distanceKm * 1);
          const speedBonus = (deliveryTime < spoilageWindow * 0.8) ? 5 : 0;
          const expected = baseCredits + distanceBonus + speedBonus;
          
          expect(credits).toBe(expected);
        }
      ),
      { numRuns: 100 }
    );
  });
});

// Feature: food-donation-matching, Property 23: Delivery Status State Machine
describe('Property: Delivery Status State Machine', () => {
  it('should only allow valid status transitions', () => {
    fc.assert(
      fc.property(
        fc.constantFrom(
          'available', 'claimed', 'assigned', 'en_route_pickup',
          'picking_up', 'en_route_delivery', 'delivering', 'delivered'
        ),
        fc.constantFrom(
          'available', 'claimed', 'assigned', 'en_route_pickup',
          'picking_up', 'en_route_delivery', 'delivering', 'delivered'
        ),
        (fromStatus, toStatus) => {
          const validTransitions = {
            'available': ['claimed'],
            'claimed': ['assigned', 'available'],
            'assigned': ['en_route_pickup'],
            'en_route_pickup': ['picking_up'],
            'picking_up': ['en_route_delivery'],
            'en_route_delivery': ['delivering'],
            'delivering': ['delivered'],
            'delivered': [],
          };
          
          const result = transitionStatus(fromStatus, toStatus);
          
          if (validTransitions[fromStatus].includes(toStatus)) {
            expect(result.success).toBe(true);
          } else {
            expect(result.success).toBe(false);
            expect(result.error).toBe('INVALID_TRANSITION');
          }
        }
      ),
      { numRuns: 100 }
    );
  });
});
```

### Integration Testing

**Test Scenarios**:
1. End-to-end delivery flow: posting → claim → assignment → pickup → delivery
2. Emergency re-matching: late cancellation → expanded radius → new claim
3. Volunteer reliability: multiple deliveries → score changes → suspension
4. Offline sync: network loss → cached operations → reconnection → sync
5. Multi-city isolation: users in different cities don't interact

**Tools**:
- Supertest for API testing
- Testcontainers for database/Redis instances
- Mock servers for external APIs
- WebSocket test clients for real-time features

### Performance Testing

**Load Testing Scenarios**:
- 1000 concurrent users creating postings
- 10,000 receivers receiving simultaneous notifications
- 500 volunteers updating location every 30 seconds
- Peak load: 100 postings/minute, 500 claims/minute

**Performance Targets**:
- API response time: p95 < 200ms, p99 < 500ms
- Notification delivery: < 30 seconds for 95% of notifications
- Geospatial queries: < 100ms for radius searches
- WebSocket message latency: < 100ms

**Tools**:
- k6 or Artillery for load testing
- Prometheus + Grafana for monitoring
- Jaeger for distributed tracing

### Security Testing

**Test Areas**:
- Authentication bypass attempts
- Authorization boundary testing
- SQL injection and NoSQL injection
- XSS and CSRF protection
- Rate limiting effectiveness
- Location data access control
- Encryption at rest verification

**Tools**:
- OWASP ZAP for vulnerability scanning
- Burp Suite for manual testing
- npm audit / pip-audit for dependency vulnerabilities

### Continuous Integration

**CI Pipeline**:
1. Lint and format check
2. Unit tests (parallel execution)
3. Property tests (100 iterations minimum)
4. Integration tests
5. Security scanning
6. Build and containerize
7. Deploy to staging
8. Smoke tests on staging

**Quality Gates**:
- All tests must pass
- Code coverage > 80%
- No high-severity security vulnerabilities
- Performance benchmarks within 10% of baseline

