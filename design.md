# SmartMandi AI - System Design Document

## Document Information
- **Project**: SmartMandi AI
- **Version**: 1.0
- **Last Updated**: February 5, 2026
- **Status**: Design Phase
- **Architecture Type**: Cloud-Native, Serverless, Event-Driven

---

## 1. High-Level Architecture Overview

SmartMandi AI is built on a fully serverless AWS architecture designed for scalability, cost-efficiency, and reliability. The system follows a microservices pattern with event-driven data processing pipelines.

### Architecture Principles

1. **Serverless-First**: Minimize operational overhead using Lambda, API Gateway, and managed services
2. **Event-Driven**: Asynchronous processing using EventBridge, SNS, and SQS
3. **Decoupled Components**: Loose coupling between data ingestion, ML, and delivery layers
4. **Scalability**: Auto-scaling at every layer to handle 100K+ farmers
5. **Cost-Optimized**: Pay-per-use model with intelligent resource allocation
6. **Security-First**: Encryption at rest and in transit, IAM least privilege
7. **Observable**: Comprehensive logging, monitoring, and alerting

### Key System Layers


| Layer | Purpose | AWS Services |
|-------|---------|--------------|
| **Data Ingestion** | Collect mandi prices and weather data | Lambda, EventBridge, S3 |
| **Data Processing** | ETL, feature engineering, validation | Lambda, Glue (optional), S3 |
| **ML Pipeline** | Model training and inference | SageMaker, S3, Lambda |
| **API Layer** | REST APIs for frontend and integrations | API Gateway, Lambda |
| **Notification Layer** | SMS and IVR delivery | SNS, Connect, Lambda |
| **Frontend** | Web dashboard | Amplify, CloudFront, S3 |
| **Authentication** | User management | Cognito |
| **Data Storage** | Persistent storage | S3, DynamoDB |
| **Monitoring** | Observability and alerting | CloudWatch, X-Ray |

---

## 2. Architecture Diagram

### System Architecture (ASCII)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           EXTERNAL DATA SOURCES                              │
│                  Agmarknet API  │  Weather API  │  Public Data               │
└────────────────────────────────┬────────────────────────────────────────────┘
                                 │
                    ┌────────────▼────────────┐
                    │   EventBridge Scheduler │
                    │   (Daily Triggers)      │
                    └────────────┬────────────┘
                                 │
                    ┌────────────▼────────────┐
                    │  Lambda: Data Ingestion │
                    │  (Fetch & Validate)     │
                    └────────────┬────────────┘
                                 │
                    ┌────────────▼────────────┐
                    │   S3: Raw Data Lake     │
                    │   /raw/prices/YYYY/MM/  │
                    └────────────┬────────────┘
                                 │
                    ┌────────────▼────────────┐
                    │  Lambda: ETL Pipeline   │
                    │  (Clean & Transform)    │
                    └────────────┬────────────┘
                                 │
                    ┌────────────▼────────────┐
                    │  S3: Processed Data     │
                    │  /processed/features/   │
                    └────────────┬────────────┘
                                 │
         ┌───────────────────────┴───────────────────────┐
         │                                               │
┌────────▼────────┐                           ┌─────────▼─────────┐
│  SageMaker      │                           │  Lambda: Batch    │
│  Training Jobs  │                           │  Inference        │
│  (Weekly)       │                           │  (Daily)          │
└────────┬────────┘                           └─────────┬─────────┘
         │                                               │
┌────────▼────────┐                           ┌─────────▼─────────┐
│  S3: ML Models  │                           │  DynamoDB:        │
│  /models/v*/    │                           │  Predictions      │
└────────┬────────┘                           └─────────┬─────────┘
         │                                               │
         └───────────────────────┬───────────────────────┘
                                 │
                    ┌────────────▼────────────┐
                    │  Lambda: Recommendation │
                    │  Engine                 │
                    └────────────┬────────────┘
                                 │
         ┌───────────────────────┴───────────────────────┐
         │                       │                       │
┌────────▼────────┐   ┌─────────▼─────────┐   ┌────────▼────────┐
│  SNS: SMS       │   │  Amazon Connect   │   │  API Gateway    │
│  Notifications  │   │  IVR System       │   │  REST APIs      │
└─────────────────┘   └───────────────────┘   └────────┬────────┘
                                                        │
                                               ┌────────▼────────┐
                                               │  Lambda:        │
                                               │  API Handlers   │
                                               └────────┬────────┘
                                                        │
                                               ┌────────▼────────┐
                                               │  Cognito:       │
                                               │  Auth           │
                                               └────────┬────────┘
                                                        │
                                               ┌────────▼────────┐
                                               │  Amplify:       │
                                               │  Web Dashboard  │
                                               │  (CloudFront)   │
                                               └─────────────────┘
                                                        │
                                               ┌────────▼────────┐
                                               │  DynamoDB:      │
                                               │  User Profiles  │
                                               └─────────────────┘

┌─────────────────────────────────────────────────────────────────────────────┐
│                    MONITORING & LOGGING (CloudWatch + X-Ray)                 │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Data Flow Diagram

```
[Agmarknet] ──┐
              ├──> [Ingestion Lambda] ──> [S3 Raw] ──> [ETL Lambda] ──> [S3 Processed]
[Weather API]─┘                                                              │
                                                                             │
                                                                             ▼
                                                        [SageMaker Training] ──> [S3 Models]
                                                                             │
                                                                             ▼
                                                        [Inference Lambda] ──> [DynamoDB Predictions]
                                                                             │
                                                                             ▼
                                                        [Recommendation Engine]
                                                                             │
                                    ┌────────────────────┬───────────────────┴──────────┐
                                    ▼                    ▼                              ▼
                                [SMS/SNS]          [IVR/Connect]                  [API Gateway]
                                    │                    │                              │
                                    ▼                    ▼                              ▼
                                [Farmers]            [Farmers]                    [Web Dashboard]
```

---

## 3. Component Design

### 3.1 Data Ingestion Layer

#### Lambda: Data Ingestion Function
- **Purpose**: Fetch daily mandi prices and weather data from external APIs
- **Trigger**: EventBridge scheduled rule (daily at 8 PM IST)
- **Runtime**: Python 3.11
- **Memory**: 512 MB
- **Timeout**: 5 minutes
- **Concurrency**: Reserved concurrency of 5

**Responsibilities**:
- Call Agmarknet API to fetch daily prices
- Call weather API for district-level data
- Validate data schema and completeness
- Handle API rate limits and retries
- Store raw data in S3 with partitioning

**Error Handling**:
- Retry logic with exponential backoff
- Dead letter queue (SQS) for failed ingestions
- CloudWatch alarms for consecutive failures

#### S3: Data Lake
**Bucket Structure**:
```
smartmandi-data-lake/
├── raw/
│   ├── prices/
│   │   └── year=2026/month=02/day=05/prices.json
│   └── weather/
│       └── year=2026/month=02/day=05/weather.json
├── processed/
│   ├── features/
│   │   └── year=2026/month=02/features.parquet
│   └── aggregated/
│       └── monthly_stats.parquet
├── models/
│   ├── v1/
│   │   ├── model.tar.gz
│   │   └── metadata.json
│   └── v2/
└── predictions/
    └── year=2026/month=02/day=05/predictions.json
```

**Lifecycle Policies**:
- Raw data: Standard → Glacier after 90 days
- Processed data: Standard → IA after 30 days
- Models: Standard (all versions retained)
- Predictions: Standard → Delete after 90 days

### 3.2 ETL Processing Layer

#### Lambda: ETL Pipeline
- **Purpose**: Clean, transform, and engineer features from raw data
- **Trigger**: S3 event notification on raw data upload
- **Runtime**: Python 3.11
- **Memory**: 1024 MB
- **Timeout**: 10 minutes

**Processing Steps**:
1. **Data Cleaning**:
   - Remove duplicates
   - Handle missing values (forward fill, interpolation)
   - Detect and cap outliers (IQR method)
   - Standardize commodity names

2. **Feature Engineering**:
   - Rolling averages (7-day, 14-day, 30-day)
   - Price volatility metrics
   - Year-over-year growth rates
   - Seasonal indicators
   - Weather features (temperature, rainfall)
   - Day-of-week and month indicators

3. **Data Validation**:
   - Schema validation using Pydantic
   - Data quality checks (completeness, accuracy)
   - Statistical anomaly detection

4. **Output**:
   - Store processed features in Parquet format
   - Trigger SageMaker training (weekly) or inference (daily)

### 3.3 ML Pipeline (SageMaker)

#### SageMaker Training Pipeline

**Training Job Configuration**:
- **Instance Type**: ml.m5.xlarge (cost-optimized)
- **Framework**: PyTorch or TensorFlow
- **Frequency**: Weekly (Sunday 2 AM IST)
- **Training Data**: Last 3 years of processed features

**Model Architecture**:
```python
# LSTM-based time series forecasting
Input: [batch_size, sequence_length=30, features=15]
├── LSTM Layer 1 (128 units)
├── Dropout (0.2)
├── LSTM Layer 2 (64 units)
├── Dropout (0.2)
├── Dense Layer (32 units, ReLU)
└── Output Layer (7 units) # 7-day forecast
```

**Training Pipeline Steps**:
1. Load processed features from S3
2. Split data: 80% train, 10% validation, 10% test
3. Hyperparameter tuning using SageMaker Automatic Model Tuning
4. Train model with early stopping
5. Evaluate on test set (MAPE, RMSE, MAE)
6. Register model in SageMaker Model Registry
7. Deploy to endpoint if accuracy > threshold

**Model Versioning**:
- All models stored in S3 with version tags
- Metadata includes: accuracy metrics, training date, hyperparameters
- A/B testing capability for model comparison

#### SageMaker Inference

**Real-time Endpoint**:
- **Instance Type**: ml.t2.medium (1 instance)
- **Auto-scaling**: Scale to 3 instances if invocations > 1000/min
- **Model**: Latest approved model from registry

**Batch Transform**:
- **Purpose**: Daily batch predictions for all crop-mandi combinations
- **Schedule**: Daily at 6 AM IST
- **Input**: Latest 30 days of features from S3
- **Output**: 7-day price forecasts to S3 and DynamoDB

### 3.4 API Layer

#### API Gateway Configuration
- **Type**: REST API
- **Authorization**: Cognito User Pools
- **Throttling**: 1000 requests/second, 5000 burst
- **Caching**: Enabled with 5-minute TTL for GET requests
- **CORS**: Enabled for Amplify domain

#### Lambda: API Handlers
- **Runtime**: Python 3.11
- **Memory**: 256 MB
- **Timeout**: 30 seconds
- **Environment Variables**: DynamoDB table names, S3 bucket

**Handler Functions**:
1. `get_price_forecast`: Retrieve predictions for crop/mandi
2. `get_recommendations`: Get sell/hold advice
3. `get_user_profile`: Fetch user preferences
4. `update_user_profile`: Update crops, mandis, language
5. `get_historical_prices`: Historical data for charts
6. `register_user`: New farmer registration

### 3.5 Notification Layer

#### Amazon SNS (SMS)
- **Purpose**: Send daily price alerts to farmers
- **Message Format**: 
  ```
  SmartMandi: [Crop] price at [Mandi] expected to [rise/fall] by [X]% in 7 days. 
  Recommendation: [SELL NOW / HOLD]. Top mandis: [M1, M2, M3]
  ```
- **Delivery**: Batch processing via Lambda
- **Cost Optimization**: 
  - Send only significant price changes (>5%)
  - Consolidate multiple crops in one message
  - Use long codes for bulk SMS

#### Amazon Connect (IVR)
- **Purpose**: Voice-based price information in regional languages
- **Flow Design**:
  ```
  1. Welcome message (language selection)
  2. Enter crop code (DTMF input)
  3. Play price forecast (text-to-speech)
  4. Play recommendation (sell/hold)
  5. Play top 3 mandis
  6. Repeat or exit
  ```
- **Integration**: Lambda function to fetch data from DynamoDB
- **Languages**: Hindi, English, Marathi, Punjabi, Telugu
- **Text-to-Speech**: Amazon Polly with regional voices

#### Lambda: Notification Orchestrator
- **Purpose**: Determine which farmers to notify and via which channel
- **Trigger**: EventBridge schedule (daily 7 AM IST)
- **Logic**:
  1. Query DynamoDB for active users
  2. Fetch predictions for user's crops
  3. Apply notification rules (price change threshold)
  4. Batch users by notification preference
  5. Invoke SNS for SMS users
  6. Queue IVR calls for voice users

### 3.6 Frontend Layer

#### AWS Amplify
- **Hosting**: Static site hosting with CloudFront CDN
- **Framework**: React.js (or Next.js for SSR)
- **Build**: Automated CI/CD from Git repository
- **Environment**: Production and staging environments

**Dashboard Features**:
- User authentication (Cognito)
- Price forecast charts (Chart.js or Recharts)
- Historical price trends
- Recommendation cards
- User profile management
- Multi-language support

**Performance Optimization**:
- Code splitting and lazy loading
- Image optimization
- Service worker for offline capability
- Gzip compression
- CloudFront caching

#### Amazon Cognito
- **User Pool**: Manage farmer accounts
- **Authentication**: Phone number + OTP
- **MFA**: Optional SMS-based MFA
- **User Attributes**: phone, language, crops, mandis, location
- **Groups**: farmers, extension_officers, admins

### 3.7 Data Storage

#### DynamoDB Tables

**Table 1: UserProfiles**
```
Partition Key: userId (phone number)
Attributes:
- phoneNumber (string)
- language (string)
- crops (list of strings)
- mandis (list of strings)
- location (string)
- notificationPreference (SMS/IVR/Both)
- notificationTime (string)
- isActive (boolean)
- createdAt (timestamp)
- lastLoginAt (timestamp)

GSI: location-index (for regional queries)
```

**Table 2: Predictions**
```
Partition Key: cropMandi (string, e.g., "wheat_delhi")
Sort Key: date (string, YYYY-MM-DD)
Attributes:
- crop (string)
- mandi (string)
- currentPrice (number)
- forecastDay1 to forecastDay7 (numbers)
- confidence (number)
- recommendation (SELL/HOLD)
- topMandis (list)
- createdAt (timestamp)

TTL: 90 days
GSI: date-index (for batch queries)
```

**Table 3: NotificationLog**
```
Partition Key: userId
Sort Key: timestamp
Attributes:
- notificationType (SMS/IVR)
- status (sent/failed)
- message (string)
- deliveryStatus (string)

TTL: 30 days
```

**Capacity Planning**:
- On-demand billing for unpredictable traffic
- Provisioned capacity for predictable patterns (cost savings)
- Auto-scaling enabled: 5-100 RCU/WCU

---

## 4. Data Flow

### 4.1 Daily Data Ingestion Flow

```
1. EventBridge triggers Ingestion Lambda at 8 PM IST
2. Lambda fetches data from Agmarknet and Weather APIs
3. Raw data validated and stored in S3 (/raw/prices/, /raw/weather/)
4. S3 event triggers ETL Lambda
5. ETL Lambda processes data and stores in S3 (/processed/features/)
6. S3 event triggers Batch Inference Lambda
7. Inference Lambda calls SageMaker endpoint
8. Predictions stored in DynamoDB and S3
9. Notification Orchestrator Lambda triggered at 7 AM next day
10. SMS/IVR notifications sent to farmers
```

### 4.2 Weekly Model Training Flow

```
1. EventBridge triggers Training Lambda on Sunday 2 AM
2. Training Lambda prepares dataset from S3 processed data
3. SageMaker Training Job launched
4. Model trained and evaluated
5. If accuracy > threshold:
   - Model registered in Model Registry
   - New endpoint created
   - Traffic shifted to new endpoint (blue-green deployment)
6. Old endpoint deleted after 24 hours
7. CloudWatch metrics logged
```

### 4.3 User Interaction Flow (Web Dashboard)

```
1. User accesses dashboard (Amplify/CloudFront)
2. Cognito authentication (phone + OTP)
3. Frontend calls API Gateway
4. API Gateway validates JWT token
5. Lambda handler queries DynamoDB for user profile
6. Lambda fetches predictions from DynamoDB
7. Response returned to frontend
8. Dashboard renders charts and recommendations
```

### 4.4 SMS Notification Flow

```
1. Notification Orchestrator Lambda triggered daily
2. Query DynamoDB for users with SMS preference
3. For each user:
   - Fetch predictions for user's crops
   - Apply notification rules (price change > 5%)
   - Format SMS message
   - Publish to SNS topic
4. SNS delivers SMS to phone number
5. Delivery status logged in DynamoDB
6. Failed deliveries sent to DLQ for retry
```

### 4.5 IVR Call Flow

```
1. Farmer dials toll-free number
2. Amazon Connect answers call
3. Language selection prompt (DTMF)
4. Crop selection prompt (DTMF)
5. Lambda function invoked with crop code
6. Lambda queries DynamoDB for predictions
7. Response returned to Connect
8. Polly converts text to speech
9. Forecast and recommendation played
10. Call ends or repeats
```

---

## 5. ML Pipeline Design

### 5.1 Training Pipeline Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                    TRAINING PIPELINE                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  [S3 Processed Data] ──> [Data Preparation Lambda]              │
│                                │                                 │
│                                ▼                                 │
│                    [SageMaker Processing Job]                    │
│                    - Feature scaling                             │
│                    - Train/val/test split                        │
│                    - Data augmentation                           │
│                                │                                 │
│                                ▼                                 │
│                    [SageMaker Training Job]                      │
│                    - Hyperparameter tuning                       │
│                    - Model training (LSTM)                       │
│                    - Cross-validation                            │
│                                │                                 │
│                                ▼                                 │
│                    [Model Evaluation]                            │
│                    - MAPE, RMSE, MAE                             │
│                    - Accuracy by crop                            │
│                    - Confidence intervals                        │
│                                │                                 │
│                    ┌───────────┴───────────┐                    │
│                    ▼                       ▼                     │
│            [Accuracy > 85%?]       [Accuracy < 85%]              │
│                    │                       │                     │
│                    ▼                       ▼                     │
│        [Model Registry]          [Retrain with tuning]           │
│        [Deploy Endpoint]         [Alert Data Science Team]       │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 5.2 Model Training Configuration

**Algorithm**: LSTM (Long Short-Term Memory) for time series forecasting

**Input Features** (15 features):
1. Historical prices (7-day, 14-day, 30-day moving averages)
2. Price volatility (standard deviation)
3. Year-over-year growth rate
4. Month-over-month growth rate
5. Day of week (one-hot encoded)
6. Month (one-hot encoded)
7. Seasonal indicator (Rabi/Kharif)
8. Temperature (average, min, max)
9. Rainfall (mm)
10. Humidity (%)
11. Supply indicator (arrivals at mandi)
12. Demand indicator (dispatches from mandi)
13. Festival indicator (binary)
14. Previous day price
15. Lag features (t-1, t-7, t-30)

**Hyperparameters**:
```python
{
    "sequence_length": 30,  # 30 days of history
    "forecast_horizon": 7,  # 7 days ahead
    "lstm_units_1": 128,
    "lstm_units_2": 64,
    "dropout_rate": 0.2,
    "learning_rate": 0.001,
    "batch_size": 32,
    "epochs": 100,
    "early_stopping_patience": 10
}
```

**Training Strategy**:
- **Per-crop models**: Separate model for each major crop (5 models)
- **Transfer learning**: Pre-train on all crops, fine-tune per crop
- **Ensemble**: Average predictions from LSTM, Prophet, and ARIMA

### 5.3 Inference Pipeline

**Real-time Inference** (for API calls):
```
API Request ──> Lambda ──> SageMaker Endpoint ──> Response
Latency: < 2 seconds
```

**Batch Inference** (for daily predictions):
```
EventBridge ──> Lambda ──> SageMaker Batch Transform ──> S3/DynamoDB
Duration: ~30 minutes for all crop-mandi combinations
```

**Inference Optimization**:
- Model quantization for faster inference
- Batch predictions cached in DynamoDB
- Real-time endpoint for on-demand queries
- Multi-model endpoint for cost savings

### 5.4 Model Monitoring

**Metrics Tracked**:
- Prediction accuracy (MAPE) per crop
- Inference latency (p50, p95, p99)
- Model drift detection (data distribution changes)
- Feature importance tracking

**Retraining Triggers**:
- Weekly scheduled retraining
- Accuracy drops below 80%
- Significant data drift detected
- New crop season begins

---

## 6. API Design

### 6.1 API Endpoints

**Base URL**: `https://api.smartmandi.ai/v1`

#### Authentication Endpoints

```
POST /auth/register
Request:
{
  "phoneNumber": "+919876543210",
  "language": "hi",
  "crops": ["wheat", "rice"],
  "mandis": ["delhi", "meerut"],
  "location": "delhi"
}
Response:
{
  "userId": "user_123",
  "message": "OTP sent to phone",
  "otpExpiry": "2026-02-05T10:05:00Z"
}

POST /auth/verify-otp
Request:
{
  "phoneNumber": "+919876543210",
  "otp": "123456"
}
Response:
{
  "accessToken": "eyJhbGc...",
  "refreshToken": "eyJhbGc...",
  "expiresIn": 3600
}

POST /auth/refresh
Request:
{
  "refreshToken": "eyJhbGc..."
}
Response:
{
  "accessToken": "eyJhbGc...",
  "expiresIn": 3600
}
```

#### Forecast Endpoints

```
GET /forecasts/{crop}/{mandi}
Headers: Authorization: Bearer <token>
Response:
{
  "crop": "wheat",
  "mandi": "delhi",
  "currentPrice": 2500,
  "unit": "quintal",
  "forecast": [
    {"day": 1, "price": 2520, "confidence": 0.92},
    {"day": 2, "price": 2545, "confidence": 0.89},
    ...
    {"day": 7, "price": 2610, "confidence": 0.78}
  ],
  "recommendation": {
    "action": "HOLD",
    "reason": "Price expected to rise by 4.4% in 7 days",
    "confidence": 0.85
  },
  "topMandis": [
    {"name": "meerut", "price": 2630, "distance": 70},
    {"name": "panipat", "price": 2615, "distance": 90},
    {"name": "karnal", "price": 2605, "distance": 120}
  ],
  "lastUpdated": "2026-02-05T08:00:00Z"
}

GET /forecasts/user
Headers: Authorization: Bearer <token>
Response:
{
  "forecasts": [
    {
      "crop": "wheat",
      "mandi": "delhi",
      "currentPrice": 2500,
      "forecast7Day": 2610,
      "priceChange": "+4.4%",
      "recommendation": "HOLD"
    },
    ...
  ]
}
```

#### Historical Data Endpoints

```
GET /historical/{crop}/{mandi}?days=90
Headers: Authorization: Bearer <token>
Response:
{
  "crop": "wheat",
  "mandi": "delhi",
  "data": [
    {"date": "2025-11-07", "price": 2300},
    {"date": "2025-11-08", "price": 2310},
    ...
  ],
  "statistics": {
    "min": 2200,
    "max": 2600,
    "average": 2450,
    "volatility": 0.08
  }
}
```

#### User Profile Endpoints

```
GET /users/profile
Headers: Authorization: Bearer <token>
Response:
{
  "userId": "user_123",
  "phoneNumber": "+919876543210",
  "language": "hi",
  "crops": ["wheat", "rice"],
  "mandis": ["delhi", "meerut"],
  "location": "delhi",
  "notificationPreference": "SMS",
  "notificationTime": "07:00"
}

PUT /users/profile
Headers: Authorization: Bearer <token>
Request:
{
  "crops": ["wheat", "rice", "cotton"],
  "mandis": ["delhi", "meerut", "panipat"],
  "notificationPreference": "BOTH"
}
Response:
{
  "message": "Profile updated successfully"
}
```

#### Admin Endpoints

```
GET /admin/metrics
Headers: Authorization: Bearer <admin-token>
Response:
{
  "totalUsers": 10000,
  "activeUsers": 8500,
  "totalPredictions": 50000,
  "averageAccuracy": 0.87,
  "smsDeliveryRate": 0.96,
  "ivrCompletionRate": 0.72
}

GET /admin/model-performance
Headers: Authorization: Bearer <admin-token>
Response:
{
  "models": [
    {
      "crop": "wheat",
      "version": "v2.1",
      "accuracy": 0.89,
      "lastTrained": "2026-02-02T02:00:00Z"
    },
    ...
  ]
}
```

### 6.2 API Rate Limiting

| User Type | Rate Limit | Burst |
|-----------|------------|-------|
| Farmer | 100 req/min | 200 |
| Extension Officer | 500 req/min | 1000 |
| Admin | 1000 req/min | 2000 |
| Anonymous | 10 req/min | 20 |

### 6.3 Error Responses

```json
{
  "error": {
    "code": "RESOURCE_NOT_FOUND",
    "message": "Forecast not available for this crop-mandi combination",
    "details": "No data available for crop: banana, mandi: shimla",
    "timestamp": "2026-02-05T10:30:00Z"
  }
}
```

**Error Codes**:
- `400`: Bad Request
- `401`: Unauthorized
- `403`: Forbidden
- `404`: Resource Not Found
- `429`: Too Many Requests
- `500`: Internal Server Error
- `503`: Service Unavailable

---

## 7. Database & Data Storage Design

### 7.1 Storage Strategy

| Data Type | Storage | Rationale | Cost/Month |
|-----------|---------|-----------|------------|
| Raw mandi prices | S3 Standard → Glacier | Infrequent access after 90 days | $5 |
| Processed features | S3 Standard → IA | Used for training, accessed weekly | $3 |
| ML models | S3 Standard | Frequent access for inference | $2 |
| Predictions | DynamoDB + S3 | Fast queries + archival | $10 |
| User profiles | DynamoDB | Low latency reads/writes | $5 |
| Logs | CloudWatch | 30-day retention | $8 |
| **Total** | | | **$33** |

### 7.2 DynamoDB Design Patterns

#### Access Patterns

1. **Get user profile by phone number** → UserProfiles (PK: userId)
2. **Get all users in a location** → UserProfiles GSI (location-index)
3. **Get prediction for crop-mandi-date** → Predictions (PK: cropMandi, SK: date)
4. **Get all predictions for a date** → Predictions GSI (date-index)
5. **Get notification history for user** → NotificationLog (PK: userId, SK: timestamp)

#### Indexing Strategy

**UserProfiles Table**:
- Primary Index: userId (phone number)
- GSI-1: location-index (for regional queries)
- GSI-2: isActive-index (for active user queries)

**Predictions Table**:
- Primary Index: cropMandi + date
- GSI-1: date-index (for batch queries)
- TTL: 90 days (automatic deletion)

#### Capacity Planning

**UserProfiles** (10,000 users):
- Item size: ~1 KB
- Total size: 10 MB
- Read pattern: 100 reads/sec (peak)
- Write pattern: 10 writes/sec
- **Capacity**: 5 RCU, 5 WCU (on-demand)

**Predictions** (50 crops × 100 mandis × 90 days):
- Item count: 450,000
- Item size: ~2 KB
- Total size: 900 MB
- Read pattern: 500 reads/sec (peak)
- Write pattern: 5000 writes/day (batch)
- **Capacity**: On-demand billing

### 7.3 S3 Bucket Organization

```
smartmandi-data-lake/
├── raw/
│   ├── prices/
│   │   └── year=2026/month=02/day=05/
│   │       ├── agmarknet_prices.json
│   │       └── metadata.json
│   └── weather/
│       └── year=2026/month=02/day=05/
│           └── weather_data.json
├── processed/
│   ├── features/
│   │   └── year=2026/month=02/
│   │       └── features_20260205.parquet
│   └── aggregated/
│       ├── monthly_stats.parquet
│       └── yearly_stats.parquet
├── models/
│   ├── wheat/
│   │   ├── v1/
│   │   │   ├── model.tar.gz
│   │   │   ├── metadata.json
│   │   │   └── metrics.json
│   │   └── v2/
│   ├── rice/
│   └── cotton/
├── predictions/
│   └── year=2026/month=02/day=05/
│       └── predictions.json
└── logs/
    └── ingestion/
        └── year=2026/month=02/day=05/
            └── ingestion.log
```

### 7.4 Data Retention Policy

| Data Type | Retention | Archive After | Delete After |
|-----------|-----------|---------------|--------------|
| Raw prices | 3 years | 90 days (Glacier) | 3 years |
| Processed features | 1 year | 30 days (IA) | 1 year |
| ML models | Indefinite | N/A | Never |
| Predictions | 90 days | N/A | 90 days (TTL) |
| User profiles | Active users | N/A | 1 year inactive |
| Notification logs | 30 days | N/A | 30 days (TTL) |
| CloudWatch logs | 30 days | N/A | 30 days |

### 7.5 Backup Strategy

**DynamoDB**:
- Point-in-time recovery (PITR) enabled
- Daily automated backups
- 30-day backup retention
- Cross-region backup for disaster recovery

**S3**:
- Versioning enabled for models bucket
- Cross-region replication for critical data
- MFA delete for production buckets

---

## 8. Notification & IVR Design

### 8.1 SMS Notification Architecture

```
┌─────────────────────────────────────────────────────────────┐
│              SMS NOTIFICATION PIPELINE                       │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  [EventBridge: Daily 7 AM] ──> [Notification Orchestrator]  │
│                                         │                    │
│                                         ▼                    │
│                          [Query DynamoDB: Active Users]      │
│                                         │                    │
│                                         ▼                    │
│                          [Filter by Notification Rules]      │
│                          - Price change > 5%                 │
│                          - User preference = SMS/Both        │
│                                         │                    │
│                                         ▼                    │
│                          [Batch Users (100 per batch)]       │
│                                         │                    │
│                                         ▼                    │
│                          [Format SMS Messages]               │
│                          - Localize to user language         │
│                          - Truncate to 160 chars             │
│                                         │                    │
│                                         ▼                    │
│                          [Publish to SNS Topic]              │
│                                         │                    │
│                          ┌──────────────┴──────────────┐     │
│                          ▼                             ▼     │
│                    [SMS Delivered]              [SMS Failed] │
│                          │                             │     │
│                          ▼                             ▼     │
│                [Log to DynamoDB]              [Send to DLQ]  │
│                                                       │      │
│                                                       ▼      │
│                                              [Retry Logic]   │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### 8.2 SMS Message Templates

**Template 1: Price Rise Alert (Hindi)**
```
SmartMandi: गेहूं दिल्ली मंडी में 7 दिन में 4.4% बढ़ने की संभावना। 
सिफारिश: रुकें। शीर्ष मंडी: मेरठ (₹2630), पानीपत (₹2615)
```

**Template 2: Price Fall Alert (English)**
```
SmartMandi: Cotton price at Nagpur expected to fall 3.2% in 7 days. 
Recommendation: SELL NOW. Top mandis: Akola (₹5800), Wardha (₹5750)
```

**Template 3: Neutral Alert**
```
SmartMandi: Rice price stable at Amritsar. No action needed. 
Call 1800-XXX-XXXX for details.
```

### 8.3 IVR System Design

#### Amazon Connect Flow

```
┌─────────────────────────────────────────────────────────────┐
│                    IVR CALL FLOW                             │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  [Farmer Dials 1800-XXX-XXXX]                                │
│           │                                                  │
│           ▼                                                  │
│  [Welcome Message]                                           │
│  "SmartMandi में आपका स्वागत है"                             │
│           │                                                  │
│           ▼                                                  │
│  [Language Selection]                                        │
│  "Press 1 for Hindi, 2 for English, 3 for Marathi..."       │
│           │                                                  │
│           ▼                                                  │
│  [Invoke Lambda: Get User Profile]                           │
│  - Lookup by caller ID                                       │
│  - Fetch user's crops                                        │
│           │                                                  │
│           ▼                                                  │
│  [Crop Selection Menu]                                       │
│  "Press 1 for Wheat, 2 for Rice, 3 for Cotton..."           │
│           │                                                  │
│           ▼                                                  │
│  [Invoke Lambda: Get Forecast]                               │
│  - Query DynamoDB for predictions                            │
│  - Format response text                                      │
│           │                                                  │
│           ▼                                                  │
│  [Play Forecast (Polly TTS)]                                 │
│  "गेहूं की कीमत दिल्ली मंडी में आज ₹2500 है..."              │
│           │                                                  │
│           ▼                                                  │
│  [Play Recommendation]                                       │
│  "हमारी सिफारिश: 7 दिन रुकें, कीमत बढ़ने की संभावना है"      │
│           │                                                  │
│           ▼                                                  │
│  [Play Top Mandis]                                           │
│  "सर्वोत्तम मंडी: मेरठ ₹2630, पानीपत ₹2615..."               │
│           │                                                  │
│           ▼                                                  │
│  [Repeat or Exit]                                            │
│  "Press 1 to repeat, 2 for another crop, 9 to exit"         │
│           │                                                  │
│           ▼                                                  │
│  [Thank You & Disconnect]                                    │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

#### Lambda Function for IVR

```python
def lambda_handler(event, context):
    """
    Handle IVR requests from Amazon Connect
    """
    # Extract parameters from Connect
    crop = event['Details']['Parameters']['crop']
    mandi = event['Details']['Parameters']['mandi']
    language = event['Details']['Parameters']['language']
    
    # Query DynamoDB for prediction
    prediction = get_prediction(crop, mandi)
    
    # Format response in selected language
    response_text = format_ivr_response(prediction, language)
    
    return {
        'statusCode': 200,
        'body': {
            'currentPrice': prediction['currentPrice'],
            'forecast': prediction['forecast'][6]['price'],  # Day 7
            'recommendation': prediction['recommendation']['action'],
            'topMandis': prediction['topMandis'][:3],
            'responseText': response_text
        }
    }
```

#### Text-to-Speech Configuration

**Amazon Polly Voices**:
- Hindi: Aditi (female, Indian accent)
- English: Raveena (female, Indian accent)
- Marathi: Aditi (supports Marathi)
- Punjabi: Custom voice (if available)
- Telugu: Custom voice (if available)

**Speech Synthesis Markup Language (SSML)**:
```xml
<speak>
    <prosody rate="slow">
        गेहूं की कीमत दिल्ली मंडी में आज 
        <say-as interpret-as="currency">₹2500</say-as> है।
        <break time="500ms"/>
        7 दिन में कीमत 
        <say-as interpret-as="currency">₹2610</say-as> 
        तक बढ़ने की संभावना है।
    </prosody>
</speak>
```

### 8.4 Notification Rules Engine

**Rule 1: Significant Price Change**
```python
if abs(price_change_percent) > 5:
    send_notification = True
```

**Rule 2: Recommendation Change**
```python
if previous_recommendation != current_recommendation:
    send_notification = True
```

**Rule 3: User Preference**
```python
if user.notification_time == current_hour:
    send_notification = True
```

**Rule 4: Frequency Limit**
```python
if last_notification_time < (current_time - 24_hours):
    send_notification = True
```

### 8.5 Cost Optimization for Notifications

**SMS Cost Reduction**:
- Send only actionable alerts (price change > 5%)
- Consolidate multiple crops in one message
- Use long codes instead of short codes
- Negotiate bulk SMS rates with provider

**IVR Cost Reduction**:
- Inbound calls only (farmer-initiated)
- Keep call duration < 2 minutes
- Cache frequently accessed data
- Use efficient TTS (Polly standard voices)

**Estimated Costs** (10,000 farmers):
- SMS: 10,000 × ₹0.20 × 30 days = ₹60,000/month
- IVR: 2,000 calls × ₹0.50 × 2 min = ₹2,000/month
- **Total**: ₹62,000/month (₹6.20 per farmer)

---

## 9. Scalability Strategy

### 9.1 Horizontal Scalability

| Component | Scaling Strategy | Max Capacity |
|-----------|------------------|--------------|
| Lambda Functions | Auto-scale (1000 concurrent) | Unlimited |
| API Gateway | Auto-scale | 10,000 req/sec |
| SageMaker Endpoint | Auto-scale (1-10 instances) | 10 instances |
| DynamoDB | On-demand / Auto-scaling | Unlimited |
| S3 | Unlimited | Unlimited |
| SNS | Auto-scale | 30,000 SMS/sec |
| Amazon Connect | Auto-scale | 1,000 concurrent calls |

### 9.2 Performance Optimization

#### Lambda Optimization
- **Provisioned Concurrency**: 5 instances for critical functions
- **Memory Allocation**: Right-sized (256 MB - 1024 MB)
- **Cold Start Mitigation**: Keep functions warm with EventBridge pings
- **Code Optimization**: Minimize dependencies, use Lambda layers

#### API Gateway Optimization
- **Caching**: 5-minute TTL for GET requests
- **Compression**: Enable gzip compression
- **Throttling**: Per-user rate limits
- **Regional Endpoints**: Deploy in ap-south-1 (Mumbai)

#### DynamoDB Optimization
- **DAX (DynamoDB Accelerator)**: Microsecond latency for hot data
- **Global Tables**: Multi-region replication for DR
- **Efficient Queries**: Use GSIs for access patterns
- **Batch Operations**: BatchGetItem, BatchWriteItem

#### SageMaker Optimization
- **Model Compilation**: Use SageMaker Neo for faster inference
- **Multi-Model Endpoints**: Host multiple models on one endpoint
- **Batch Transform**: For bulk predictions
- **Elastic Inference**: Attach GPU acceleration only when needed

### 9.3 Caching Strategy

```
┌─────────────────────────────────────────────────────────────┐
│                    CACHING LAYERS                            │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  [CloudFront] ──> Static assets (1 day TTL)                  │
│       │                                                      │
│       ▼                                                      │
│  [API Gateway Cache] ──> API responses (5 min TTL)           │
│       │                                                      │
│       ▼                                                      │
│  [DynamoDB DAX] ──> Hot predictions (1 min TTL)              │
│       │                                                      │
│       ▼                                                      │
│  [Lambda Memory] ──> ML models, config (function lifetime)   │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

**Cache Invalidation**:
- CloudFront: Invalidate on new deployment
- API Gateway: Invalidate on new predictions
- DAX: Automatic TTL-based expiration

### 9.4 Load Testing Plan

**Test Scenarios**:
1. **Normal Load**: 100 req/sec, 10,000 users
2. **Peak Load**: 500 req/sec, 50,000 users
3. **Spike Load**: 1000 req/sec for 5 minutes
4. **Sustained Load**: 200 req/sec for 24 hours

**Tools**:
- AWS Load Testing Solution
- Apache JMeter
- Locust.io

**Metrics to Monitor**:
- API latency (p50, p95, p99)
- Lambda throttles and errors
- DynamoDB throttles
- SageMaker endpoint latency
- Error rates

### 9.5 Scaling Triggers

**Auto-scaling Policies**:

**SageMaker Endpoint**:
```python
{
    "TargetValue": 70.0,  # Target invocations per instance
    "ScaleInCooldown": 300,  # 5 minutes
    "ScaleOutCooldown": 60   # 1 minute
}
```

**DynamoDB**:
```python
{
    "TargetUtilization": 70.0,  # Target utilization %
    "ScaleInCooldown": 60,
    "ScaleOutCooldown": 60,
    "MinCapacity": 5,
    "MaxCapacity": 100
}
```

**Lambda Reserved Concurrency**:
- Critical functions: 50 reserved
- Non-critical: Unreserved (shared pool)

---

## 10. Security Design

### 10.1 Security Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    SECURITY LAYERS                           │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  [WAF] ──> DDoS protection, IP filtering, rate limiting      │
│     │                                                        │
│     ▼                                                        │
│  [CloudFront] ──> HTTPS only, signed URLs                    │
│     │                                                        │
│     ▼                                                        │
│  [API Gateway] ──> Cognito auth, API keys, throttling        │
│     │                                                        │
│     ▼                                                        │
│  [Lambda] ──> IAM roles, VPC (optional), secrets manager     │
│     │                                                        │
│     ▼                                                        │
│  [DynamoDB/S3] ──> Encryption at rest, VPC endpoints         │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### 10.2 Authentication & Authorization

#### Cognito User Pool Configuration
```json
{
  "MfaConfiguration": "OPTIONAL",
  "PasswordPolicy": {
    "MinimumLength": 8,
    "RequireUppercase": false,
    "RequireLowercase": false,
    "RequireNumbers": true,
    "RequireSymbols": false
  },
  "UserAttributeUpdateSettings": {
    "AttributesRequireVerificationBeforeUpdate": ["phone_number"]
  },
  "AccountRecoverySetting": {
    "RecoveryMechanisms": [
      {"Name": "verified_phone_number", "Priority": 1}
    ]
  }
}
```

#### IAM Roles & Policies

**Lambda Execution Role**:
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "logs:CreateLogGroup",
        "logs:CreateLogStream",
        "logs:PutLogEvents"
      ],
      "Resource": "arn:aws:logs:*:*:*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "dynamodb:GetItem",
        "dynamodb:PutItem",
        "dynamodb:Query"
      ],
      "Resource": "arn:aws:dynamodb:ap-south-1:*:table/SmartMandi*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject"
      ],
      "Resource": "arn:aws:s3:::smartmandi-data-lake/*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "sagemaker:InvokeEndpoint"
      ],
      "Resource": "arn:aws:sagemaker:ap-south-1:*:endpoint/smartmandi-*"
    }
  ]
}
```

**Least Privilege Principle**: Each Lambda function has minimal permissions

### 10.3 Data Encryption

#### Encryption at Rest
- **S3**: SSE-S3 (AES-256) or SSE-KMS for sensitive data
- **DynamoDB**: AWS-managed keys (default) or customer-managed KMS keys
- **EBS Volumes**: Encrypted (SageMaker training instances)
- **Secrets Manager**: Encrypted API keys and credentials

#### Encryption in Transit
- **HTTPS/TLS 1.2+**: All API communications
- **VPC Endpoints**: Private connectivity to AWS services
- **Certificate Management**: AWS Certificate Manager (ACM)

### 10.4 Network Security

#### VPC Configuration (Optional for Lambda)
```
VPC: smartmandi-vpc (10.0.0.0/16)
├── Public Subnets (10.0.1.0/24, 10.0.2.0/24)
│   └── NAT Gateways
└── Private Subnets (10.0.10.0/24, 10.0.11.0/24)
    └── Lambda Functions (if VPC-enabled)
    └── VPC Endpoints (S3, DynamoDB, SageMaker)
```

**Security Groups**:
- Lambda SG: Outbound only (HTTPS to AWS services)
- VPC Endpoints: Inbound from Lambda SG

#### AWS WAF Rules
1. **Rate Limiting**: Max 100 req/5min per IP
2. **Geo-blocking**: Allow only India traffic
3. **SQL Injection Protection**: Block common patterns
4. **XSS Protection**: Block script tags
5. **Known Bad IPs**: Block AWS-managed threat list

### 10.5 Secrets Management

**AWS Secrets Manager**:
- Agmarknet API credentials
- Weather API keys
- Database connection strings
- Third-party service tokens

**Rotation Policy**: 90 days for API keys

**Access Pattern**:
```python
import boto3
from botocore.exceptions import ClientError

def get_secret(secret_name):
    client = boto3.client('secretsmanager', region_name='ap-south-1')
    try:
        response = client.get_secret_value(SecretId=secret_name)
        return response['SecretString']
    except ClientError as e:
        # Handle error
        raise e
```

### 10.6 Compliance & Auditing

**CloudTrail**:
- Enable for all regions
- Log API calls to S3
- Integrate with CloudWatch for alerting

**AWS Config**:
- Track resource configuration changes
- Compliance rules for encryption, public access

**GuardDuty**:
- Threat detection for AWS accounts
- Monitor for suspicious activity

**Compliance Requirements**:
- IT Act 2000 (India)
- TRAI regulations for SMS/voice
- Data localization (store data in India region)

### 10.7 Security Checklist

- [ ] All S3 buckets have encryption enabled
- [ ] All S3 buckets block public access
- [ ] DynamoDB tables use encryption at rest
- [ ] API Gateway uses Cognito authorization
- [ ] Lambda functions use IAM roles (no hardcoded credentials)
- [ ] Secrets stored in Secrets Manager
- [ ] CloudTrail enabled for audit logging
- [ ] WAF rules configured for API Gateway
- [ ] HTTPS enforced for all endpoints
- [ ] Regular security audits scheduled
- [ ] Vulnerability scanning enabled
- [ ] Incident response plan documented

---

## 11. Cost Optimization Strategy

### 11.1 Cost Breakdown (Monthly Estimate for 10,000 Users)

| Service | Usage | Cost |
|---------|-------|------|
| **Lambda** | 10M invocations, 512 MB, 3 sec avg | $15 |
| **API Gateway** | 5M requests | $18 |
| **SageMaker Training** | 4 hours/week, ml.m5.xlarge | $25 |
| **SageMaker Inference** | 1 endpoint, ml.t2.medium | $35 |
| **S3** | 100 GB storage, 10 GB transfer | $5 |
| **DynamoDB** | On-demand, 10M reads, 1M writes | $15 |
| **SNS (SMS)** | 300,000 messages @ $0.00645 | $1,935 |
| **Amazon Connect** | 2,000 calls, 2 min avg | $20 |
| **Cognito** | 10,000 MAU | $27.50 |
| **CloudWatch** | Logs, metrics, alarms | $10 |
| **Amplify** | Hosting, build minutes | $5 |
| **Data Transfer** | 50 GB outbound | $4.50 |
| **Total** | | **$2,115** |

**Cost per Farmer**: $2,115 / 10,000 = **$0.21/month** (₹17.50)

### 11.2 Cost Optimization Techniques

#### Compute Optimization
1. **Lambda**:
   - Right-size memory allocation
   - Use ARM64 (Graviton2) for 20% cost savings
   - Minimize cold starts with provisioned concurrency (only critical functions)
   - Reduce package size (use Lambda layers)

2. **SageMaker**:
   - Use Spot Instances for training (70% savings)
   - Schedule training during off-peak hours
   - Use multi-model endpoints
   - Consider SageMaker Serverless Inference for low traffic

#### Storage Optimization
1. **S3**:
   - Lifecycle policies (Standard → IA → Glacier)
   - Intelligent-Tiering for unpredictable access
   - Compress data (gzip, parquet)
   - Delete incomplete multipart uploads

2. **DynamoDB**:
   - Use on-demand for unpredictable traffic
   - Switch to provisioned capacity for predictable patterns
   - Enable auto-scaling
   - Archive old data to S3

#### Network Optimization
1. **Data Transfer**:
   - Use CloudFront for caching (reduce origin requests)
   - VPC endpoints for S3/DynamoDB (no data transfer charges)
   - Compress API responses

2. **API Gateway**:
   - Enable caching (reduce Lambda invocations)
   - Use HTTP API instead of REST API (60% cheaper)

#### Notification Optimization
1. **SMS**:
   - Send only actionable alerts (reduce volume by 50%)
   - Consolidate messages (multiple crops in one SMS)
   - Negotiate bulk rates with SMS provider
   - Use regional SMS providers (cheaper than SNS)

2. **IVR**:
   - Inbound only (farmer-initiated)
   - Keep calls short (< 2 minutes)
   - Cache frequently accessed data

### 11.3 Cost Monitoring

**AWS Cost Explorer**:
- Daily cost tracking
- Budget alerts ($2,500/month threshold)
- Cost allocation tags (by service, environment)

**Custom CloudWatch Metrics**:
- Cost per farmer
- Cost per prediction
- Cost per notification

**Optimization Targets**:
- Reduce cost per farmer to < ₹5/month
- Notification costs < 80% of total
- Compute costs < 15% of total

### 11.4 Reserved Capacity & Savings Plans

**Compute Savings Plan** (1-year commitment):
- Lambda: 10% savings
- SageMaker: 20% savings

**Reserved Instances**:
- SageMaker endpoint: 30% savings (1-year RI)

**Estimated Savings**: $300/month (15% reduction)

---

## 12. Deployment Plan

### 12.1 Infrastructure as Code

**AWS CDK (TypeScript)**:
```typescript
// lib/smartmandi-stack.ts
import * as cdk from 'aws-cdk-lib';
import * as lambda from 'aws-cdk-lib/aws-lambda';
import * as apigateway from 'aws-cdk-lib/aws-apigateway';
import * as dynamodb from 'aws-cdk-lib/aws-dynamodb';
import * as s3 from 'aws-cdk-lib/aws-s3';

export class SmartMandiStack extends cdk.Stack {
  constructor(scope: cdk.App, id: string, props?: cdk.StackProps) {
    super(scope, id, props);

    // S3 Data Lake
    const dataLake = new s3.Bucket(this, 'DataLake', {
      bucketName: 'smartmandi-data-lake',
      encryption: s3.BucketEncryption.S3_MANAGED,
      blockPublicAccess: s3.BlockPublicAccess.BLOCK_ALL,
      lifecycleRules: [
        {
          transitions: [
            { storageClass: s3.StorageClass.GLACIER, transitionAfter: cdk.Duration.days(90) }
          ]
        }
      ]
    });

    // DynamoDB Tables
    const userTable = new dynamodb.Table(this, 'UserProfiles', {
      tableName: 'SmartMandi-UserProfiles',
      partitionKey: { name: 'userId', type: dynamodb.AttributeType.STRING },
      billingMode: dynamodb.BillingMode.PAY_PER_REQUEST,
      encryption: dynamodb.TableEncryption.AWS_MANAGED,
      pointInTimeRecovery: true
    });

    // Lambda Functions
    const ingestionFunction = new lambda.Function(this, 'IngestionFunction', {
      runtime: lambda.Runtime.PYTHON_3_11,
      handler: 'index.handler',
      code: lambda.Code.fromAsset('lambda/ingestion'),
      timeout: cdk.Duration.minutes(5),
      memorySize: 512,
      environment: {
        DATA_LAKE_BUCKET: dataLake.bucketName
      }
    });

    // API Gateway
    const api = new apigateway.RestApi(this, 'SmartMandiAPI', {
      restApiName: 'SmartMandi API',
      deployOptions: {
        stageName: 'prod',
        throttlingRateLimit: 1000,
        throttlingBurstLimit: 5000
      }
    });
  }
}
```

### 12.2 CI/CD Pipeline

```
┌─────────────────────────────────────────────────────────────┐
│                    CI/CD PIPELINE                            │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  [GitHub] ──> [Push to main]                                 │
│                     │                                        │
│                     ▼                                        │
│              [GitHub Actions]                                │
│                     │                                        │
│         ┌───────────┴───────────┐                            │
│         ▼                       ▼                            │
│    [Run Tests]            [Build Artifacts]                  │
│    - Unit tests           - Lambda packages                  │
│    - Integration tests    - CDK synth                        │
│    - Linting              - Docker images                    │
│         │                       │                            │
│         └───────────┬───────────┘                            │
│                     ▼                                        │
│              [Deploy to Dev]                                 │
│              - CDK deploy                                    │
│              - Smoke tests                                   │
│                     │                                        │
│                     ▼                                        │
│              [Manual Approval]                               │
│                     │                                        │
│                     ▼                                        │
│              [Deploy to Prod]                                │
│              - Blue-green deployment                         │
│              - Health checks                                 │
│              - Rollback on failure                           │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

**GitHub Actions Workflow**:
```yaml
name: Deploy SmartMandi

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - name: Set up Python
        uses: actions/setup-python@v4
        with:
          python-version: '3.11'
      - name: Install dependencies
        run: pip install -r requirements.txt
      - name: Run tests
        run: pytest tests/
      - name: Run linting
        run: flake8 src/

  deploy-dev:
    needs: test
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    steps:
      - uses: actions/checkout@v3
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v2
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ap-south-1
      - name: Deploy to Dev
        run: |
          npm install -g aws-cdk
          cdk deploy --require-approval never

  deploy-prod:
    needs: deploy-dev
    runs-on: ubuntu-latest
    environment: production
    steps:
      - uses: actions/checkout@v3
      - name: Deploy to Production
        run: cdk deploy --context env=prod
```

### 12.3 Environment Strategy

| Environment | Purpose | AWS Account | Region |
|-------------|---------|-------------|--------|
| **Dev** | Development and testing | Dev account | ap-south-1 |
| **Staging** | Pre-production validation | Dev account | ap-south-1 |
| **Prod** | Production | Prod account | ap-south-1 |
| **DR** | Disaster recovery | Prod account | ap-southeast-1 |

### 12.4 Deployment Checklist

**Pre-Deployment**:
- [ ] All tests passing
- [ ] Code review approved
- [ ] Security scan completed
- [ ] Performance testing done
- [ ] Documentation updated
- [ ] Rollback plan prepared

**Deployment**:
- [ ] Deploy to dev environment
- [ ] Run smoke tests
- [ ] Deploy to staging
- [ ] Run integration tests
- [ ] Manual QA approval
- [ ] Deploy to production (blue-green)
- [ ] Monitor metrics for 1 hour
- [ ] Switch traffic to new version

**Post-Deployment**:
- [ ] Verify all endpoints
- [ ] Check CloudWatch metrics
- [ ] Review error logs
- [ ] Test notifications (SMS/IVR)
- [ ] Update runbook
- [ ] Notify stakeholders

### 12.5 Blue-Green Deployment

**SageMaker Endpoint**:
```python
# Create new endpoint configuration
new_endpoint_config = sagemaker.create_endpoint_config(
    EndpointConfigName=f'smartmandi-config-v{new_version}',
    ProductionVariants=[
        {
            'VariantName': 'AllTraffic',
            'ModelName': f'smartmandi-model-v{new_version}',
            'InitialInstanceCount': 1,
            'InstanceType': 'ml.t2.medium'
        }
    ]
)

# Update endpoint with new config
sagemaker.update_endpoint(
    EndpointName='smartmandi-endpoint',
    EndpointConfigName=f'smartmandi-config-v{new_version}'
)
```

**Lambda Alias**:
```python
# Publish new version
new_version = lambda_client.publish_version(
    FunctionName='smartmandi-api-handler'
)

# Update alias to point to new version
lambda_client.update_alias(
    FunctionName='smartmandi-api-handler',
    Name='prod',
    FunctionVersion=new_version['Version']
)
```

### 12.6 Rollback Strategy

**Automated Rollback Triggers**:
- Error rate > 5% for 5 minutes
- Latency p99 > 5 seconds
- SageMaker endpoint health check fails

**Manual Rollback**:
```bash
# Rollback Lambda
aws lambda update-alias \
  --function-name smartmandi-api-handler \
  --name prod \
  --function-version <previous-version>

# Rollback SageMaker endpoint
aws sagemaker update-endpoint \
  --endpoint-name smartmandi-endpoint \
  --endpoint-config-name smartmandi-config-v<previous-version>

# Rollback CDK stack
cdk deploy --rollback
```

---

## 13. Monitoring & Logging

### 13.1 Observability Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                  OBSERVABILITY STACK                         │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  [Application Logs] ──> [CloudWatch Logs]                    │
│                              │                               │
│                              ▼                               │
│                    [Log Insights Queries]                    │
│                    [Metric Filters]                          │
│                              │                               │
│                              ▼                               │
│  [Metrics] ──> [CloudWatch Metrics] ──> [Dashboards]         │
│                              │                               │
│                              ▼                               │
│                    [CloudWatch Alarms]                       │
│                              │                               │
│                              ▼                               │
│                    [SNS Notifications]                       │
│                              │                               │
│                    ┌─────────┴─────────┐                    │
│                    ▼                   ▼                     │
│              [Email Alerts]      [Slack Webhook]             │
│                                                              │
│  [Traces] ──> [X-Ray] ──> [Service Map]                      │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### 13.2 CloudWatch Dashboards

**Dashboard 1: System Health**
- API Gateway request count, latency, errors
- Lambda invocations, duration, errors, throttles
- SageMaker endpoint invocations, latency
- DynamoDB read/write capacity, throttles

**Dashboard 2: Business Metrics**
- Total active users
- Daily predictions generated
- SMS delivery rate
- IVR call completion rate
- Forecast accuracy (MAPE)

**Dashboard 3: Cost Metrics**
- Daily AWS spend by service
- Cost per farmer
- Cost per prediction
- Cost per notification

### 13.3 Key Metrics

#### Application Metrics

| Metric | Threshold | Alarm |
|--------|-----------|-------|
| API Latency (p99) | < 2 seconds | > 3 seconds |
| API Error Rate | < 1% | > 5% |
| Lambda Error Rate | < 0.5% | > 2% |
| Lambda Throttles | 0 | > 10 |
| SageMaker Latency | < 2 seconds | > 5 seconds |
| DynamoDB Throttles | 0 | > 5 |
| SMS Delivery Rate | > 95% | < 90% |
| IVR Completion Rate | > 70% | < 60% |

#### Business Metrics

| Metric | Target | Alert |
|--------|--------|-------|
| Daily Active Users | 8,000 | < 5,000 |
| Forecast Accuracy (MAPE) | < 15% | > 20% |
| User Satisfaction (NPS) | > 50 | < 30 |
| Notification Open Rate | > 60% | < 40% |

### 13.4 Logging Strategy

**Log Levels**:
- **ERROR**: System errors, exceptions
- **WARN**: Degraded performance, retries
- **INFO**: Business events (user registration, predictions)
- **DEBUG**: Detailed debugging (dev only)

**Structured Logging** (JSON format):
```json
{
  "timestamp": "2026-02-05T10:30:00Z",
  "level": "INFO",
  "service": "api-handler",
  "function": "get_forecast",
  "requestId": "abc-123",
  "userId": "user_123",
  "crop": "wheat",
  "mandi": "delhi",
  "latency": 245,
  "message": "Forecast retrieved successfully"
}
```

**Log Retention**:
- Production: 30 days
- Development: 7 days
- Archived logs: S3 (1 year)

**Log Insights Queries**:

**Query 1: Error Rate by Function**
```
fields @timestamp, @message
| filter level = "ERROR"
| stats count() by function
| sort count desc
```

**Query 2: Slow API Requests**
```
fields @timestamp, userId, crop, mandi, latency
| filter latency > 2000
| sort latency desc
| limit 20
```

**Query 3: Failed Notifications**
```
fields @timestamp, userId, notificationType, status
| filter status = "failed"
| stats count() by notificationType
```

### 13.5 Distributed Tracing (X-Ray)

**Instrumentation**:
```python
from aws_xray_sdk.core import xray_recorder
from aws_xray_sdk.core import patch_all

# Patch all supported libraries
patch_all()

@xray_recorder.capture('get_forecast')
def get_forecast(crop, mandi):
    # Add metadata
    xray_recorder.put_metadata('crop', crop)
    xray_recorder.put_metadata('mandi', mandi)
    
    # Add annotation (indexed)
    xray_recorder.put_annotation('crop_type', crop)
    
    # Function logic
    prediction = query_dynamodb(crop, mandi)
    return prediction
```

**Trace Analysis**:
- End-to-end request flow
- Service dependencies
- Bottleneck identification
- Error root cause analysis

### 13.6 Alerting Strategy

**Critical Alerts** (PagerDuty):
- API Gateway 5xx errors > 10 in 5 minutes
- Lambda function errors > 50 in 5 minutes
- SageMaker endpoint down
- DynamoDB throttling > 100 in 5 minutes

**Warning Alerts** (Slack):
- API latency p99 > 2 seconds
- SMS delivery rate < 95%
- Forecast accuracy < 85%
- Daily cost > $100

**Info Alerts** (Email):
- Daily summary report
- Weekly model performance report
- Monthly cost report

---

## 14. Failure Handling & Reliability

### 14.1 Failure Modes & Mitigation

| Failure Mode | Impact | Probability | Mitigation |
|--------------|--------|-------------|------------|
| Agmarknet API down | No new data | Medium | Cache last 7 days, retry logic, backup source |
| Lambda timeout | Failed requests | Low | Increase timeout, optimize code, async processing |
| SageMaker endpoint down | No predictions | Low | Multi-endpoint, cached predictions, fallback model |
| DynamoDB throttling | Slow responses | Medium | Auto-scaling, on-demand billing, caching |
| SMS delivery failure | Missed notifications | Medium | Retry logic, DLQ, alternative channels |
| S3 outage | Data unavailable | Very Low | Cross-region replication, local caching |

### 14.2 Retry Logic

**Exponential Backoff**:
```python
import time
from functools import wraps

def retry_with_backoff(max_retries=3, base_delay=1):
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            for attempt in range(max_retries):
                try:
                    return func(*args, **kwargs)
                except Exception as e:
                    if attempt == max_retries - 1:
                        raise
                    delay = base_delay * (2 ** attempt)
                    time.sleep(delay)
            return None
        return wrapper
    return decorator

@retry_with_backoff(max_retries=3, base_delay=2)
def fetch_agmarknet_data():
    # API call logic
    pass
```

### 14.3 Circuit Breaker Pattern

```python
class CircuitBreaker:
    def __init__(self, failure_threshold=5, timeout=60):
        self.failure_count = 0
        self.failure_threshold = failure_threshold
        self.timeout = timeout
        self.last_failure_time = None
        self.state = 'CLOSED'  # CLOSED, OPEN, HALF_OPEN
    
    def call(self, func, *args, **kwargs):
        if self.state == 'OPEN':
            if time.time() - self.last_failure_time > self.timeout:
                self.state = 'HALF_OPEN'
            else:
                raise Exception("Circuit breaker is OPEN")
        
        try:
            result = func(*args, **kwargs)
            self.on_success()
            return result
        except Exception as e:
            self.on_failure()
            raise e
    
    def on_success(self):
        self.failure_count = 0
        self.state = 'CLOSED'
    
    def on_failure(self):
        self.failure_count += 1
        self.last_failure_time = time.time()
        if self.failure_count >= self.failure_threshold:
            self.state = 'OPEN'
```

### 14.4 Dead Letter Queues

**SQS DLQ Configuration**:
```python
# Create DLQ
dlq = sqs.create_queue(
    QueueName='smartmandi-dlq',
    Attributes={
        'MessageRetentionPeriod': '1209600'  # 14 days
    }
)

# Configure main queue with DLQ
main_queue = sqs.create_queue(
    QueueName='smartmandi-notifications',
    Attributes={
        'RedrivePolicy': json.dumps({
            'deadLetterTargetArn': dlq['QueueArn'],
            'maxReceiveCount': '3'
        })
    }
)
```

**DLQ Processing**:
- Manual review of failed messages
- Automated retry after fixing root cause
- Alert on DLQ depth > 100

### 14.5 Disaster Recovery

**RTO (Recovery Time Objective)**: 4 hours  
**RPO (Recovery Point Objective)**: 1 hour

**Backup Strategy**:
- DynamoDB: Point-in-time recovery (PITR)
- S3: Cross-region replication to ap-southeast-1
- Lambda: Code in Git, infrastructure in CDK
- SageMaker models: Versioned in S3

**DR Runbook**:
1. Detect outage (CloudWatch alarms)
2. Assess impact and scope
3. Activate DR plan
4. Restore from backups
5. Switch DNS to DR region (Route 53)
6. Verify functionality
7. Communicate with users
8. Post-mortem analysis

### 14.6 Health Checks

**API Health Endpoint**:
```python
@app.route('/health')
def health_check():
    checks = {
        'dynamodb': check_dynamodb(),
        'sagemaker': check_sagemaker_endpoint(),
        's3': check_s3_access(),
        'timestamp': datetime.utcnow().isoformat()
    }
    
    all_healthy = all(checks.values())
    status_code = 200 if all_healthy else 503
    
    return jsonify(checks), status_code
```

**Synthetic Monitoring**:
- CloudWatch Synthetics canaries
- Test critical user journeys every 5 minutes
- Alert on failures

---

## 15. Trade-offs & Design Decisions

### 15.1 Serverless vs. Container-based

**Decision**: Serverless (Lambda, SageMaker Serverless)

**Rationale**:
- **Pros**:
  - Zero operational overhead
  - Auto-scaling built-in
  - Pay-per-use (cost-effective for variable load)
  - Fast iteration and deployment
  - Ideal for event-driven architecture
  
- **Cons**:
  - Cold start latency (mitigated with provisioned concurrency)
  - 15-minute Lambda timeout (acceptable for our use case)
  - Vendor lock-in (acceptable for hackathon/MVP)

**Alternative Considered**: ECS Fargate
- More control, but higher operational complexity
- Not justified for MVP scale

### 15.2 Real-time vs. Batch Inference

**Decision**: Hybrid approach
- **Batch inference** for daily predictions (all crop-mandi combinations)
- **Real-time endpoint** for on-demand API queries

**Rationale**:
- Batch is cost-effective for bulk predictions
- Real-time provides flexibility for API users
- Cached batch predictions serve 95% of requests
- Real-time handles edge cases and new queries

**Trade-off**: Slight increase in complexity, but significant cost savings

### 15.3 DynamoDB vs. RDS

**Decision**: DynamoDB for user profiles and predictions

**Rationale**:
- **Pros**:
  - Serverless, auto-scaling
  - Single-digit millisecond latency
  - No database administration
  - Built-in backup and PITR
  - Cost-effective for key-value access patterns
  
- **Cons**:
  - Limited query flexibility (mitigated with GSIs)
  - No complex joins (not needed for our schema)

**Alternative Considered**: Aurora Serverless
- Better for complex queries, but higher cost and operational overhead
- Overkill for simple key-value access patterns

### 15.4 LSTM vs. Prophet vs. ARIMA

**Decision**: LSTM as primary model, with ensemble option

**Rationale**:
- **LSTM**:
  - Handles non-linear patterns
  - Captures long-term dependencies
  - Works well with multivariate features (weather, seasonality)
  - Industry-proven for time series
  
- **Prophet** (Facebook):
  - Good for seasonal patterns
  - Easier to interpret
  - Faster training
  - Used as ensemble member
  
- **ARIMA**:
  - Classical approach
  - Works for stationary series
  - Less flexible than LSTM

**Trade-off**: LSTM requires more data and compute, but provides better accuracy

### 15.5 SMS vs. WhatsApp vs. Mobile App

**Decision**: SMS + IVR for MVP, WhatsApp for Phase 2

**Rationale**:
- **SMS**:
  - Universal (works on all phones)
  - No internet required
  - High delivery rate
  - Regulatory compliance easier
  
- **WhatsApp**:
  - Richer content (images, buttons)
  - Lower cost per message
  - Requires smartphone and internet
  - WhatsApp Business API has approval delays
  
- **Mobile App**:
  - Best user experience
  - Requires smartphone and app store presence
  - Higher development cost
  - Deferred to Phase 3

**Trade-off**: SMS is more expensive but reaches more farmers

### 15.6 Multi-region vs. Single-region

**Decision**: Single region (ap-south-1 Mumbai) for MVP, DR in ap-southeast-1

**Rationale**:
- **Single Region**:
  - Simpler architecture
  - Lower cost
  - Faster development
  - Data localization compliance (India)
  
- **Multi-region**:
  - Better disaster recovery
  - Lower latency for global users (not needed)
  - Higher complexity and cost

**Trade-off**: Acceptable risk for MVP, DR region provides backup

### 15.7 Monolithic API vs. Microservices

**Decision**: Monolithic Lambda functions with logical separation

**Rationale**:
- **Monolithic**:
  - Simpler deployment
  - Shared code and dependencies
  - Faster development for small team
  - Lower cold start overhead
  
- **Microservices**:
  - Better separation of concerns
  - Independent scaling
  - More complex deployment
  - Overkill for MVP

**Trade-off**: Start monolithic, refactor to microservices if needed

### 15.8 Synchronous vs. Asynchronous Processing

**Decision**: Asynchronous for data ingestion and training, synchronous for API

**Rationale**:
- **Async** (EventBridge, SQS):
  - Decouples components
  - Better fault tolerance
  - Handles spikes gracefully
  - Used for: data ingestion, ETL, training, notifications
  
- **Sync** (API Gateway):
  - Immediate response
  - Simpler error handling
  - Used for: user-facing APIs

**Trade-off**: Increased complexity, but better reliability

### 15.9 Caching Strategy

**Decision**: Multi-layer caching (CloudFront, API Gateway, DAX)

**Rationale**:
- Reduces backend load by 80%
- Improves latency significantly
- Cost-effective (cache hits are cheap)
- Predictions don't change frequently (daily updates)

**Trade-off**: Cache invalidation complexity, but manageable with TTLs

### 15.10 Authentication: Phone OTP vs. Email/Password

**Decision**: Phone number + OTP

**Rationale**:
- Phone number is primary identifier for farmers
- No need to remember passwords
- SMS OTP is familiar to Indian users
- Aligns with notification channel (SMS)

**Trade-off**: SMS cost for OTP, but minimal (₹0.20 per registration)

---

## 16. Future Enhancements & Roadmap

### Phase 1: MVP (Hackathon - 2 weeks)
- ✅ Data ingestion pipeline (Agmarknet + Weather)
- ✅ Basic LSTM model for 5 crops
- ✅ SMS notifications
- ✅ Simple IVR (Hindi + English)
- ✅ Web dashboard (React)
- ✅ User registration and authentication

### Phase 2: Post-Hackathon (1-3 months)
- Expand to 20 crops and 200 mandis
- Add 5 regional languages
- Implement ensemble models (LSTM + Prophet)
- WhatsApp Business API integration
- Admin analytics dashboard
- Farmer feedback mechanism
- A/B testing framework

### Phase 3: Scale (3-6 months)
- Progressive Web App (PWA) with offline support
- Personalized recommendations (ML-based)
- Community features (farmer forums)
- Integration with government schemes
- Transportation cost optimization
- Crop advisory based on predicted prices
- Mobile app (Android)

### Phase 4: Advanced Features (6-12 months)
- AI chatbot for farmer queries
- Satellite imagery for crop health
- Supply chain optimization
- Predictive analytics for crop selection
- Blockchain for price transparency
- Integration with digital payment systems
- Export market price integration
- Climate change impact modeling

---

## 17. Appendix

### 17.1 Technology Stack Summary

| Layer | Technology | Version |
|-------|------------|---------|
| **Cloud Provider** | AWS | - |
| **Compute** | Lambda | Python 3.11 |
| **API** | API Gateway | REST API |
| **ML** | SageMaker | PyTorch 2.0 |
| **Storage** | S3, DynamoDB | - |
| **Notifications** | SNS, Connect | - |
| **Frontend** | React, Amplify | React 18 |
| **Auth** | Cognito | - |
| **IaC** | AWS CDK | TypeScript |
| **CI/CD** | GitHub Actions | - |
| **Monitoring** | CloudWatch, X-Ray | - |

### 17.2 AWS Service Limits

| Service | Default Limit | Required | Action |
|---------|---------------|----------|--------|
| Lambda Concurrent Executions | 1,000 | 1,000 | OK |
| API Gateway Rate | 10,000 req/sec | 1,000 req/sec | OK |
| SageMaker Endpoints | 10 | 5 | OK |
| SNS SMS (India) | 10 SMS/sec | 100 SMS/sec | Request increase |
| DynamoDB Tables | 256 | 5 | OK |
| S3 Buckets | 100 | 3 | OK |

### 17.3 Glossary

- **Mandi**: Agricultural market (APMC - Agricultural Produce Market Committee)
- **Quintal**: Unit of weight (100 kg)
- **Rabi/Kharif**: Crop seasons in India
- **MAPE**: Mean Absolute Percentage Error (forecast accuracy metric)
- **IVR**: Interactive Voice Response
- **DTMF**: Dual-Tone Multi-Frequency (phone keypad input)
- **TTS**: Text-to-Speech
- **GSI**: Global Secondary Index (DynamoDB)
- **TTL**: Time to Live
- **PITR**: Point-in-Time Recovery

### 17.4 References

1. **AWS Documentation**:
   - [Lambda Best Practices](https://docs.aws.amazon.com/lambda/latest/dg/best-practices.html)
   - [SageMaker Developer Guide](https://docs.aws.amazon.com/sagemaker/latest/dg/)
   - [API Gateway Documentation](https://docs.aws.amazon.com/apigateway/)

2. **Data Sources**:
   - [Agmarknet](https://agmarknet.gov.in)
   - [India Meteorological Department](https://mausam.imd.gov.in)

3. **ML Resources**:
   - [Time Series Forecasting with LSTM](https://arxiv.org/abs/1909.00590)
   - [Prophet: Forecasting at Scale](https://facebook.github.io/prophet/)

4. **AWS Architecture**:
   - [AWS Well-Architected Framework](https://aws.amazon.com/architecture/well-architected/)
   - [Serverless Application Lens](https://docs.aws.amazon.com/wellarchitected/latest/serverless-applications-lens/)

### 17.5 Contact & Support

- **Project Repository**: https://github.com/smartmandi/smartmandi-ai
- **Documentation**: https://docs.smartmandi.ai
- **Support Email**: support@smartmandi.ai
- **Farmer Helpline**: 1800-XXX-XXXX (Toll-free)

---

## Document Revision History

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 2026-02-05 | SmartMandi Team | Initial design document |

---

**End of Design Document**

*This document is a living document and will be updated as the system evolves.*
