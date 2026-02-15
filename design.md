# System Design Document: Procurement Transparency AI

## Executive Summary

This document specifies the technical architecture for an AI-powered public procurement transparency platform. The system ingests structured procurement data, applies statistical and rule-based risk detection algorithms, and presents findings through an explainable web dashboard. The design prioritizes transparency, scalability, and ethical safeguards while remaining feasible for hackathon implementation.

**Core Value Proposition**: Transform opaque procurement data into actionable transparency insights without making accusations.

**Technical Philosophy**: Explainable algorithms over black-box AI, citizen usability over feature complexity, ethical safeguards by design.

## System Architecture

### High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                         Client Layer                             │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐          │
│  │   Web App    │  │  Mobile Web  │  │  API Clients │          │
│  │   (React)    │  │  (Responsive)│  │  (Future)    │          │
│  └──────────────┘  └──────────────┘  └──────────────┘          │
└─────────────────────────────────────────────────────────────────┘
                              │
                              │ HTTPS/REST
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                      Application Layer                           │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │              API Gateway (Express/Flask)                  │  │
│  │  ┌────────────┐ ┌────────────┐ ┌────────────┐           │  │
│  │  │   Auth     │ │  Dashboard │ │    RTI     │           │  │
│  │  │  Service   │ │   Service  │ │  Generator │           │  │
│  │  └────────────┘ └────────────┘ └────────────┘           │  │
│  └──────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                      Processing Layer                            │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │           Risk Detection Engine (Python)                  │  │
│  │  ┌────────────┐ ┌────────────┐ ┌────────────┐           │  │
│  │  │  Pattern   │ │ Statistical│ │    NLP     │           │  │
│  │  │  Detector  │ │  Analyzer  │ │  Processor │           │  │
│  │  └────────────┘ └────────────┘ └────────────┘           │  │
│  │  ┌────────────────────────────────────────────┐          │  │
│  │  │      Explainability Generator              │          │  │
│  │  └────────────────────────────────────────────┘          │  │
│  └──────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                        Data Layer                                │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐          │
│  │  PostgreSQL  │  │    Redis     │  │  File Store  │          │
│  │  (Primary)   │  │   (Cache)    │  │  (CSV/PDF)   │          │
│  └──────────────┘  └──────────────┘  └──────────────┘          │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                      External Layer                              │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐          │
│  │  GEM Portal  │  │ State Tender │  │  Email SMTP  │          │
│  │  (Future)    │  │ APIs (Future)│  │   Service    │          │
│  └──────────────┘  └──────────────┘  └──────────────┘          │
└─────────────────────────────────────────────────────────────────┘
```

### Component Breakdown

#### 1. Frontend Layer (React SPA)

**Technology Stack**:
- React 18+ with functional components and hooks
- React Router for navigation
- Chart.js for time series and bar charts
- D3.js for network graphs and advanced visualizations
- Leaflet for geographic maps
- Axios for HTTP requests
- Tailwind CSS for responsive styling

**Key Components**:

```
src/
├── components/
│   ├── Dashboard/
│   │   ├── SummaryCards.jsx          # KPI metrics display
│   │   ├── FilterPanel.jsx           # Department, date, risk filters
│   │   ├── ProcurementTable.jsx      # Paginated data table
│   │   └── DetailModal.jsx           # Risk detail view
│   ├── Visualizations/
│   │   ├── TimeSeriesChart.jsx       # Risk trends over time
│   │   ├── NetworkGraph.jsx          # Vendor-department relationships
│   │   └── ChoroplethMap.jsx         # Geographic risk distribution
│   ├── RTI/
│   │   ├── RTIGenerator.jsx          # Form for RTI creation
│   │   └── RTIPreview.jsx            # PDF preview before download
│   └── Auth/
│       ├── Login.jsx
│       ├── Register.jsx
│       └── Profile.jsx
├── services/
│   ├── api.js                        # Axios instance with interceptors
│   ├── auth.js                       # JWT token management
│   └── procurement.js                # API calls for procurement data
├── hooks/
│   ├── useProcurement.js             # Data fetching hook
│   └── useAuth.js                    # Authentication state hook
└── utils/
    ├── formatters.js                 # Date, currency formatting
    └── validators.js                 # Form validation
```

**State Management**:
- React Context API for global state (auth, user preferences)
- Local component state for UI interactions
- React Query (optional) for server state caching

**Routing Structure**:
```
/                    → Landing page with overview
/dashboard           → Main procurement monitoring dashboard
/procurement/:id     → Detailed view of single procurement record
/rti/generate        → RTI request generator
/profile             → User profile and preferences
/login               → Authentication
/register            → User registration
```

#### 2. Backend Layer (Express.js / Flask)

**Technology Stack**:
- Node.js with Express.js OR Python with Flask
- JWT for authentication (jsonwebtoken / PyJWT)
- bcrypt for password hashing
- PostgreSQL client (pg / psycopg2)
- Redis client (redis / redis-py)
- Multer / Flask-Upload for file uploads

**API Architecture**:

```
backend/
├── routes/
│   ├── auth.js                       # POST /api/auth/register, /login
│   ├── procurement.js                # GET /api/procurement, /api/procurement/:id
│   ├── upload.js                     # POST /api/upload (CSV ingestion)
│   ├── rti.js                        # POST /api/rti/generate
│   └── user.js                       # GET/PUT /api/user/preferences
├── controllers/
│   ├── authController.js             # Business logic for auth
│   ├── procurementController.js      # Query building, filtering
│   └── rtiController.js              # Template population
├── middleware/
│   ├── authenticate.js               # JWT verification
│   ├── rateLimiter.js                # Rate limiting logic
│   └── errorHandler.js               # Centralized error handling
├── models/
│   ├── User.js                       # User schema/ORM
│   ├── Procurement.js                # Procurement record schema
│   └── RiskFlag.js                   # Risk detection results
├── services/
│   ├── riskDetector.js               # Interface to Python risk engine
│   ├── emailService.js               # Email sending (nodemailer)
│   └── cacheService.js               # Redis operations
└── utils/
    ├── validators.js                 # Input validation schemas
    └── logger.js                     # Winston/Bunyan logging
```

**API Endpoints**:


**Authentication & User Management**:
```
POST   /api/auth/register              # Create new user account
POST   /api/auth/login                 # Authenticate and get JWT
POST   /api/auth/logout                # Invalidate token
GET    /api/auth/verify?token={}       # Email verification
POST   /api/auth/reset-request         # Request password reset
POST   /api/auth/reset-password        # Complete password reset
GET    /api/user/profile               # Get user details
PUT    /api/user/preferences           # Update user preferences
DELETE /api/user/account               # Delete user account
```

**Procurement Data**:
```
GET    /api/procurement                # List procurement records
       Query params: 
         - department: string
         - start_date: YYYY-MM-DD
         - end_date: YYYY-MM-DD
         - min_amount: number
         - max_amount: number
         - risk_level: low|medium|high
         - page: number
         - limit: number (default 50)
       Response: {
         data: Procurement[],
         pagination: { total, page, pages },
         summary: { total_contracts, flagged_count, avg_risk_score }
       }

GET    /api/procurement/:id            # Get single procurement record
       Response: {
         procurement: Procurement,
         risk_analysis: ExplainabilityReport,
         related_contracts: Procurement[]
       }

GET    /api/procurement/stats          # Dashboard statistics
       Query params: filters (same as list)
       Response: {
         total_contracts: number,
         flagged_percentage: number,
         risk_by_department: { dept: string, score: number }[],
         risk_by_pattern: { pattern: string, count: number }[],
         time_series: { month: string, count: number }[]
       }
```

**Data Ingestion**:
```
POST   /api/upload                     # Upload CSV file
       Content-Type: multipart/form-data
       Body: { file: File }
       Response: {
         status: "success" | "partial" | "failed",
         total_rows: number,
         successful_rows: number,
         failed_rows: number,
         errors: { row: number, message: string }[]
       }

GET    /api/upload/template            # Download CSV template
       Response: CSV file with required columns
```

**RTI Generation**:
```
POST   /api/rti/generate               # Generate RTI request
       Body: {
         procurement_id: string,
         applicant_name: string,
         applicant_address: string,
         applicant_email: string,
         format: "pdf" | "txt"
       }
       Response: {
         file_url: string,
         preview_text: string
       }

GET    /api/rti/download/:id           # Download generated RTI file
       Response: PDF or TXT file
```

**Analytics**:
```
GET    /api/analytics/vendors          # Vendor analysis
       Query params: vendor_id, department
       Response: {
         vendor_id: string,
         contract_count: number,
         total_value: number,
         departments: string[],
         risk_flags: number
       }

GET    /api/analytics/trends           # Risk trend analysis
       Query params: time_period, department
       Response: {
         trend_data: { date: string, risk_score: number }[],
         pattern_distribution: { pattern: string, percentage: number }[]
       }
```

#### 3. Risk Detection Engine (Python)

**Technology Stack**:
- Python 3.9+
- pandas for data manipulation
- numpy for numerical operations
- scipy for statistical analysis
- scikit-learn for ML utilities (optional)
- spaCy or NLTK for NLP
- Jinja2 for template rendering

**Module Structure**:

```
risk_engine/
├── detectors/
│   ├── base_detector.py              # Abstract base class
│   ├── single_bidder.py              # Requirement 1 implementation
│   ├── vendor_repetition.py          # Requirement 2 implementation
│   ├── compressed_deadline.py        # Requirement 3 implementation
│   ├── budget_anomaly.py             # Requirement 4 implementation
│   └── spec_tailoring.py             # Requirement 5 implementation
├── analyzers/
│   ├── statistical_analyzer.py       # Z-score, percentile calculations
│   ├── nlp_analyzer.py               # Text pattern matching
│   └── temporal_analyzer.py          # Time-based pattern detection
├── explainability/
│   ├── report_generator.py           # ExplainabilityReport creation
│   ├── templates/                    # Jinja2 templates for explanations
│   │   ├── single_bidder.j2
│   │   ├── vendor_repetition.j2
│   │   └── ...
│   └── language_generator.py         # Plain-language summary generation
├── scoring/
│   ├── risk_scorer.py                # Combined risk score calculation
│   └── weight_config.json            # Pattern weights configuration
├── data/
│   ├── historical_baseline.py        # Historical data management
│   └── knowledge_base.json           # Brand names, restrictive patterns
└── main.py                           # Entry point for risk analysis
```

**Risk Detection Pipeline**:

```python
# Pseudocode for risk detection flow

def analyze_procurement_record(record: ProcurementRecord) -> RiskAnalysis:
    """
    Main entry point for risk analysis
    """
    risk_patterns = []
    
    # Run all detectors
    detectors = [
        SingleBidderDetector(),
        VendorRepetitionDetector(),
        CompressedDeadlineDetector(),
        BudgetAnomalyDetector(),
        SpecTailoringDetector()
    ]
    
    for detector in detectors:
        if detector.is_applicable(record):
            pattern = detector.detect(record)
            if pattern:
                risk_patterns.append(pattern)
    
    # Calculate combined risk score
    risk_score = RiskScorer.calculate_combined_score(risk_patterns)
    
    # Generate explainability report
    report = ReportGenerator.generate(
        record=record,
        patterns=risk_patterns,
        score=risk_score
    )
    
    return RiskAnalysis(
        procurement_id=record.id,
        risk_score=risk_score,
        patterns=risk_patterns,
        explainability_report=report
    )
```

**Detector Interface**:

```python
from abc import ABC, abstractmethod

class BaseDetector(ABC):
    """
    Abstract base class for all risk detectors
    """
    
    @abstractmethod
    def detect(self, record: ProcurementRecord) -> Optional[RiskPattern]:
        """
        Analyze record and return RiskPattern if detected, None otherwise
        """
        pass
    
    @abstractmethod
    def is_applicable(self, record: ProcurementRecord) -> bool:
        """
        Check if this detector applies to the given record
        """
        pass
    
    def get_weight(self) -> float:
        """
        Return weight for this pattern in combined scoring
        """
        return 1.0
```

**Example: Single Bidder Detector**:

```python
class SingleBidderDetector(BaseDetector):
    """
    Detects tenders with only one bidder (Requirement 1)
    """
    
    def is_applicable(self, record: ProcurementRecord) -> bool:
        return record.bidder_count is not None
    
    def detect(self, record: ProcurementRecord) -> Optional[RiskPattern]:
        if record.bidder_count != 1:
            return None
        
        # Calculate risk score per Requirement 1.2
        base_score = 60
        value_bonus = min(20, record.tender_value_lakhs / 50)
        high_value_bonus = 15 if record.tender_value > 1000000 else 0
        
        risk_score = base_score + value_bonus + high_value_bonus
        
        # Get expected bidder count from historical baseline
        expected_count = self._get_expected_bidder_count(
            category=record.category,
            value=record.tender_value
        )
        
        return RiskPattern(
            type="SINGLE_BIDDER",
            score=risk_score,
            evidence={
                "bidder_count": 1,
                "expected_count": expected_count,
                "tender_value": record.tender_value
            },
            explanation=self._generate_explanation(record, expected_count)
        )
    
    def _get_expected_bidder_count(self, category: str, value: float) -> float:
        # Query historical baseline
        baseline = HistoricalBaseline.get_category_stats(category)
        return baseline.avg_bidder_count
    
    def _generate_explanation(self, record: ProcurementRecord, expected: float) -> str:
        return f"This tender received only 1 bid, while similar tenders " \
               f"typically receive {expected:.1f} bids on average. " \
               f"Single-bidder situations may indicate limited competition."
```

#### 4. Database Schema (PostgreSQL)

**Schema Design**:

```sql
-- Users table
CREATE TABLE users (
    user_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email VARCHAR(255) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    full_name VARCHAR(255) NOT NULL,
    email_verified BOOLEAN DEFAULT FALSE,
    verification_token UUID,
    verification_token_expires TIMESTAMP,
    reset_token UUID,
    reset_token_expires TIMESTAMP,
    last_login_date TIMESTAMP,
    account_status VARCHAR(50) DEFAULT 'ACTIVE',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_users_verification_token ON users(verification_token);

-- User preferences
CREATE TABLE user_preferences (
    user_id UUID PRIMARY KEY REFERENCES users(user_id) ON DELETE CASCADE,
    watched_departments JSONB DEFAULT '[]',
    risk_threshold INTEGER DEFAULT 70,
    email_alerts_enabled BOOLEAN DEFAULT FALSE,
    language VARCHAR(10) DEFAULT 'en',
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Procurement records
CREATE TABLE procurement_records (
    procurement_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tender_id VARCHAR(255) NOT NULL,
    department_id VARCHAR(255) NOT NULL,
    department_name VARCHAR(500) NOT NULL,
    tender_title TEXT NOT NULL,
    estimated_budget DECIMAL(15, 2) NOT NULL,
    publication_date DATE NOT NULL,
    submission_deadline DATE NOT NULL,
    bidder_count INTEGER,
    awarded_vendor_id VARCHAR(255),
    awarded_vendor_name VARCHAR(500),
    awarded_amount DECIMAL(15, 2),
    award_date DATE,
    specifications_text TEXT,
    category VARCHAR(255),
    geographic_region VARCHAR(255),
    procurement_year INTEGER,
    data_source VARCHAR(100),
    validation_status VARCHAR(50) DEFAULT 'VALID',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    UNIQUE(tender_id, department_id)
);

CREATE INDEX idx_procurement_department_date ON procurement_records(department_id, award_date);
CREATE INDEX idx_procurement_vendor_date ON procurement_records(awarded_vendor_id, award_date);
CREATE INDEX idx_procurement_category ON procurement_records(category);
CREATE INDEX idx_procurement_year ON procurement_records(procurement_year);
CREATE INDEX idx_procurement_publication_date ON procurement_records(publication_date);

-- Risk flags
CREATE TABLE risk_flags (
    flag_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    procurement_id UUID REFERENCES procurement_records(procurement_id) ON DELETE CASCADE,
    risk_pattern VARCHAR(100) NOT NULL,
    risk_score DECIMAL(5, 2) NOT NULL,
    evidence JSONB NOT NULL,
    explanation TEXT NOT NULL,
    detected_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_risk_flags_procurement ON risk_flags(procurement_id);
CREATE INDEX idx_risk_flags_pattern ON risk_flags(risk_pattern);
CREATE INDEX idx_risk_flags_score ON risk_flags(risk_score DESC);

-- Explainability reports
CREATE TABLE explainability_reports (
    report_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    procurement_id UUID REFERENCES procurement_records(procurement_id) ON DELETE CASCADE,
    overall_risk_score DECIMAL(5, 2) NOT NULL,
    risk_patterns JSONB NOT NULL,
    summary_text TEXT NOT NULL,
    disclaimer TEXT NOT NULL,
    generated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_explainability_procurement ON explainability_reports(procurement_id);

-- Historical baseline (for statistical comparisons)
CREATE TABLE procurement_history (
    history_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    category VARCHAR(255) NOT NULL,
    geographic_region VARCHAR(255) NOT NULL,
    procurement_year INTEGER NOT NULL,
    avg_amount DECIMAL(15, 2),
    stddev_amount DECIMAL(15, 2),
    avg_bidder_count DECIMAL(5, 2),
    avg_deadline_days DECIMAL(5, 2),
    sample_size INTEGER,
    computed_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_history_category_region_year ON procurement_history(category, geographic_region, procurement_year);

-- RTI requests generated
CREATE TABLE rti_requests (
    rti_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID REFERENCES users(user_id) ON DELETE SET NULL,
    procurement_id UUID REFERENCES procurement_records(procurement_id) ON DELETE CASCADE,
    applicant_name VARCHAR(255) NOT NULL,
    applicant_address TEXT NOT NULL,
    applicant_email VARCHAR(255) NOT NULL,
    request_text TEXT NOT NULL,
    file_path VARCHAR(500),
    format VARCHAR(10),
    generated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_rti_user ON rti_requests(user_id);
CREATE INDEX idx_rti_procurement ON rti_requests(procurement_id);

-- Audit logs
CREATE TABLE audit_logs (
    log_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID REFERENCES users(user_id) ON DELETE SET NULL,
    action VARCHAR(100) NOT NULL,
    resource_type VARCHAR(100),
    resource_id UUID,
    details JSONB,
    ip_address INET,
    user_agent TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_audit_user ON audit_logs(user_id);
CREATE INDEX idx_audit_action ON audit_logs(action);
CREATE INDEX idx_audit_created ON audit_logs(created_at DESC);

-- Ingestion logs
CREATE TABLE ingestion_logs (
    log_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    filename VARCHAR(500) NOT NULL,
    total_rows INTEGER NOT NULL,
    successful_rows INTEGER NOT NULL,
    failed_rows INTEGER NOT NULL,
    error_details JSONB,
    uploaded_by UUID REFERENCES users(user_id) ON DELETE SET NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_ingestion_created ON ingestion_logs(created_at DESC);

-- Algorithm audit log
CREATE TABLE algorithm_audit_log (
    audit_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    version_number VARCHAR(50) NOT NULL,
    change_description TEXT NOT NULL,
    changed_by VARCHAR(255),
    risk_pattern_affected VARCHAR(100),
    config_snapshot JSONB,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_algorithm_audit_created ON algorithm_audit_log(created_at DESC);
```

**Database Views for Common Queries**:

```sql
-- View: Procurement records with risk scores
CREATE VIEW procurement_with_risk AS
SELECT 
    p.*,
    COALESCE(e.overall_risk_score, 0) as risk_score,
    CASE 
        WHEN e.overall_risk_score IS NULL THEN 'NOT_ANALYZED'
        WHEN e.overall_risk_score < 40 THEN 'LOW'
        WHEN e.overall_risk_score <= 70 THEN 'MEDIUM'
        ELSE 'HIGH'
    END as risk_level,
    e.summary_text as risk_summary
FROM procurement_records p
LEFT JOIN explainability_reports e ON p.procurement_id = e.procurement_id;

-- View: Department risk statistics
CREATE VIEW department_risk_stats AS
SELECT 
    department_id,
    department_name,
    COUNT(*) as total_contracts,
    COUNT(CASE WHEN risk_score > 0 THEN 1 END) as flagged_contracts,
    ROUND(AVG(risk_score), 2) as avg_risk_score,
    SUM(awarded_amount) as total_value
FROM procurement_with_risk
WHERE award_date IS NOT NULL
GROUP BY department_id, department_name;

-- View: Vendor statistics
CREATE VIEW vendor_stats AS
SELECT 
    awarded_vendor_id,
    awarded_vendor_name,
    COUNT(*) as contract_count,
    COUNT(DISTINCT department_id) as department_count,
    SUM(awarded_amount) as total_value,
    AVG(risk_score) as avg_risk_score
FROM procurement_with_risk
WHERE awarded_vendor_id IS NOT NULL
GROUP BY awarded_vendor_id, awarded_vendor_name;
```


#### 5. Caching Layer (Redis)

**Cache Strategy**:

```
# Cache keys structure
dashboard:stats:{filters_hash}          # TTL: 300s (5 min)
procurement:record:{id}                 # TTL: 600s (10 min)
procurement:list:{filters_hash}:{page}  # TTL: 300s
user:session:{user_id}                  # TTL: 604800s (7 days)
rate_limit:{ip}:{endpoint}              # TTL: 900s (15 min)
historical:baseline:{category}:{region} # TTL: 86400s (24 hours)
```

**Implementation**:

```javascript
// Node.js example
const redis = require('redis');
const client = redis.createClient({
    host: process.env.REDIS_HOST,
    port: process.env.REDIS_PORT
});

async function getCachedOrFetch(key, fetchFn, ttl = 300) {
    // Try cache first
    const cached = await client.get(key);
    if (cached) {
        return JSON.parse(cached);
    }
    
    // Fetch from database
    const data = await fetchFn();
    
    // Store in cache
    await client.setex(key, ttl, JSON.stringify(data));
    
    return data;
}

// Usage in API endpoint
app.get('/api/procurement/stats', async (req, res) => {
    const filtersHash = hashFilters(req.query);
    const cacheKey = `dashboard:stats:${filtersHash}`;
    
    const stats = await getCachedOrFetch(
        cacheKey,
        () => database.getStats(req.query),
        300
    );
    
    res.json(stats);
});
```

## Data Flow Diagrams

### 1. Data Ingestion Flow

```
User uploads CSV
       │
       ▼
┌──────────────────┐
│  API Gateway     │
│  POST /upload    │
└──────────────────┘
       │
       ▼
┌──────────────────┐
│  File Validation │
│  - Schema check  │
│  - Size limit    │
└──────────────────┘
       │
       ▼
┌──────────────────┐
│  Data Parser     │
│  - CSV to JSON   │
│  - Type casting  │
└──────────────────┘
       │
       ▼
┌──────────────────┐
│  Normalization   │
│  - Lowercase     │
│  - Date format   │
│  - Amount units  │
└──────────────────┘
       │
       ▼
┌──────────────────┐
│  Deduplication   │
│  - Check exists  │
│  - Conflict res. │
└──────────────────┘
       │
       ▼
┌──────────────────┐
│  Batch Insert    │
│  PostgreSQL      │
└──────────────────┘
       │
       ▼
┌──────────────────┐
│  Trigger Risk    │
│  Analysis Queue  │
└──────────────────┘
       │
       ▼
Return upload summary
```

### 2. Risk Analysis Flow

```
Procurement Record
       │
       ▼
┌──────────────────────────────────────┐
│     Risk Detection Engine            │
│                                      │
│  ┌────────────────────────────────┐ │
│  │  Load Historical Baseline      │ │
│  │  - Category stats              │ │
│  │  - Regional averages           │ │
│  └────────────────────────────────┘ │
│                                      │
│  ┌────────────────────────────────┐ │
│  │  Run Parallel Detectors        │ │
│  │  ├─ Single Bidder              │ │
│  │  ├─ Vendor Repetition          │ │
│  │  ├─ Compressed Deadline        │ │
│  │  ├─ Budget Anomaly             │ │
│  │  └─ Spec Tailoring             │ │
│  └────────────────────────────────┘ │
│                                      │
│  ┌────────────────────────────────┐ │
│  │  Aggregate Results             │ │
│  │  - Collect patterns            │ │
│  │  - Calculate combined score    │ │
│  └────────────────────────────────┘ │
│                                      │
│  ┌────────────────────────────────┐ │
│  │  Generate Explainability       │ │
│  │  - Template rendering          │ │
│  │  - Plain language summary      │ │
│  └────────────────────────────────┘ │
└──────────────────────────────────────┘
       │
       ▼
┌──────────────────┐
│  Store Results   │
│  - risk_flags    │
│  - explainability│
└──────────────────┘
       │
       ▼
Return RiskAnalysis
```

### 3. Dashboard Query Flow

```
User requests dashboard
       │
       ▼
┌──────────────────┐
│  API Gateway     │
│  GET /procurement│
└──────────────────┘
       │
       ▼
┌──────────────────┐
│  Check Redis     │
│  Cache           │
└──────────────────┘
       │
       ├─ Cache Hit ──────────┐
       │                      │
       └─ Cache Miss          │
              │               │
              ▼               │
       ┌──────────────────┐  │
       │  Build Query     │  │
       │  - Apply filters │  │
       │  - Pagination    │  │
       └──────────────────┘  │
              │               │
              ▼               │
       ┌──────────────────┐  │
       │  Execute Query   │  │
       │  PostgreSQL      │  │
       └──────────────────┘  │
              │               │
              ▼               │
       ┌──────────────────┐  │
       │  Store in Cache  │  │
       └──────────────────┘  │
              │               │
              └───────────────┘
                      │
                      ▼
              ┌──────────────────┐
              │  Format Response │
              │  - Pagination    │
              │  - Summary stats │
              └──────────────────┘
                      │
                      ▼
              Return JSON to client
```

### 4. RTI Generation Flow

```
User clicks "Generate RTI"
       │
       ▼
┌──────────────────┐
│  Collect Input   │
│  - Name          │
│  - Address       │
│  - Email         │
└──────────────────┘
       │
       ▼
┌──────────────────┐
│  Validate Input  │
│  - Required      │
│  - Format check  │
└──────────────────┘
       │
       ▼
┌──────────────────┐
│  Fetch Data      │
│  - Procurement   │
│  - Risk patterns │
└──────────────────┘
       │
       ▼
┌──────────────────┐
│  Select Template │
│  Based on risks  │
└──────────────────┘
       │
       ▼
┌──────────────────┐
│  Generate Qs     │
│  Context-aware   │
│  questions       │
└──────────────────┘
       │
       ▼
┌──────────────────┐
│  Render Template │
│  Jinja2/Mustache │
└──────────────────┘
       │
       ▼
┌──────────────────┐
│  Generate PDF    │
│  WeasyPrint/     │
│  ReportLab       │
└──────────────────┘
       │
       ▼
┌──────────────────┐
│  Store File      │
│  - Database ref  │
│  - File system   │
└──────────────────┘
       │
       ▼
Return download link
```

## Risk Scoring Logic Design

### Individual Pattern Scoring

**1. Single Bidder (Base: 60-95)**:
```
score = 60 + min(20, tender_value_lakhs / 50)
if tender_value > 1,000,000:
    score += 15
```

**2. Vendor Repetition (Base: 40-100)**:
```
score = min(100, 40 + (contract_count * 10) + (total_value_crores * 5))
if total_value > 10,000,000 within 180 days:
    score = max(score, 70)
```

**3. Compressed Deadline (Base: 50-100)**:
```
expected_days = category_median
deviation_ratio = (expected_days - actual_days) / expected_days
score = 50 + (deviation_ratio * 50)
```

**4. Budget Anomaly (Base: 65-100)**:
```
z_score = (awarded_amount - historical_mean) / historical_stddev
score = 65 + min(35, z_score * 10)
```

**5. Spec Tailoring (Base: 70-100)**:
```
score = 70 + min(30, tailoring_indicator_count * 5)
```

### Combined Risk Score Calculation

```python
def calculate_combined_score(patterns: List[RiskPattern]) -> float:
    """
    Calculate combined risk score with interaction effects
    """
    if not patterns:
        return 0.0
    
    # Pattern weights (configurable)
    weights = {
        'SINGLE_BIDDER': 1.0,
        'VENDOR_REPETITION': 1.2,
        'COMPRESSED_DEADLINE': 0.9,
        'BUDGET_ANOMALY': 1.1,
        'SPEC_TAILORING': 1.0
    }
    
    # Weighted sum
    weighted_sum = sum(
        pattern.score * weights.get(pattern.type, 1.0)
        for pattern in patterns
    )
    
    # Interaction multiplier (patterns co-occurring increase risk)
    interaction_multiplier = 1.0 + (0.05 * (len(patterns) - 1))
    
    # Special case: COMPRESSED_DEADLINE + SINGLE_BIDDER
    if has_patterns(patterns, ['COMPRESSED_DEADLINE', 'SINGLE_BIDDER']):
        interaction_multiplier *= 1.15
    
    # Calculate final score
    combined_score = weighted_sum * interaction_multiplier
    
    # Normalize to 0-100 range
    return min(100.0, combined_score)
```

### Risk Level Classification

```
score < 40:   LOW (Green)
40 ≤ score ≤ 70: MEDIUM (Yellow)
score > 70:   HIGH (Red)
```

## Explainability Layer Design

### Report Structure

```json
{
  "report_id": "uuid",
  "procurement_id": "uuid",
  "overall_risk_score": 78.5,
  "risk_level": "HIGH",
  "risk_patterns": [
    {
      "pattern_type": "SINGLE_BIDDER",
      "score_contribution": 65,
      "weight": 1.0,
      "evidence": {
        "bidder_count": 1,
        "expected_count": 4.2,
        "tender_value": 5000000
      },
      "explanation": "This tender received only 1 bid, while similar tenders typically receive 4.2 bids on average. Single-bidder situations may indicate limited competition."
    },
    {
      "pattern_type": "COMPRESSED_DEADLINE",
      "score_contribution": 68,
      "weight": 0.9,
      "evidence": {
        "actual_deadline_days": 5,
        "expected_deadline_days": 14,
        "deviation_percentage": 64.3
      },
      "explanation": "The bidding window of 5 days is 64% shorter than the category median of 14 days, potentially limiting vendor participation."
    }
  ],
  "interaction_effects": {
    "multiplier": 1.15,
    "reason": "COMPRESSED_DEADLINE and SINGLE_BIDDER co-occurrence increases risk"
  },
  "summary_text": "This tender shows 2 risk indicators: single bidder participation and compressed deadline. These patterns suggest potential issues requiring further investigation, not proof of wrongdoing.",
  "disclaimer": "Risk scores are analytical indicators, not evidence of wrongdoing. Further investigation is required before drawing conclusions.",
  "generated_at": "2026-02-15T10:30:00Z"
}
```

### Natural Language Generation

**Template-Based Approach**:

```python
# templates/single_bidder.j2
This tender received only {{ evidence.bidder_count }} bid, 
while similar tenders typically receive {{ evidence.expected_count | round(1) }} bids on average.
{% if evidence.tender_value > 1000000 %}
Given the high value of ₹{{ evidence.tender_value | format_currency }}, 
this warrants closer examination.
{% endif %}
Single-bidder situations may indicate limited competition or barriers to entry.

# templates/summary.j2
This tender shows {{ pattern_count }} risk indicator{% if pattern_count > 1 %}s{% endif %}: 
{{ pattern_list | join(', ') }}.
{% if risk_level == 'HIGH' %}
The high risk score suggests this procurement may benefit from detailed review.
{% elif risk_level == 'MEDIUM' %}
These patterns suggest potential issues requiring further investigation.
{% else %}
While flagged, the overall risk level is low.
{% endif %}
This analysis does not constitute proof of wrongdoing.
```

### Visualization of Risk Breakdown

**Pie Chart Data**:
```json
{
  "chart_type": "pie",
  "data": [
    { "label": "Single Bidder", "value": 65, "color": "#FF6384" },
    { "label": "Compressed Deadline", "value": 68, "color": "#36A2EB" },
    { "label": "Interaction Effect", "value": 10, "color": "#FFCE56" }
  ]
}
```

## Scalability Considerations

### Horizontal Scaling Architecture

```
                    ┌─────────────────┐
                    │  Load Balancer  │
                    │   (Nginx/ALB)   │
                    └─────────────────┘
                            │
        ┌───────────────────┼───────────────────┐
        │                   │                   │
        ▼                   ▼                   ▼
┌───────────────┐   ┌───────────────┐   ┌───────────────┐
│  API Server 1 │   │  API Server 2 │   │  API Server N │
│  (Stateless)  │   │  (Stateless)  │   │  (Stateless)  │
└───────────────┘   └───────────────┘   └───────────────┘
        │                   │                   │
        └───────────────────┼───────────────────┘
                            │
        ┌───────────────────┼───────────────────┐
        │                   │                   │
        ▼                   ▼                   ▼
┌───────────────┐   ┌───────────────┐   ┌───────────────┐
│  PostgreSQL   │   │  Redis Cache  │   │  Risk Engine  │
│  (Primary +   │   │   Cluster     │   │  Workers      │
│   Replicas)   │   │               │   │  (Queue)      │
└───────────────┘   └───────────────┘   └───────────────┘
```

### Database Optimization

**1. Indexing Strategy**:
- Composite indexes on frequently filtered columns
- Partial indexes for common query patterns
- GIN indexes for JSONB columns

**2. Query Optimization**:
```sql
-- Use EXPLAIN ANALYZE to identify slow queries
EXPLAIN ANALYZE
SELECT * FROM procurement_with_risk
WHERE department_id = 'DEPT001'
  AND award_date BETWEEN '2025-01-01' AND '2025-12-31'
  AND risk_score > 70
ORDER BY risk_score DESC
LIMIT 50;

-- Add covering index
CREATE INDEX idx_dept_date_risk_covering 
ON procurement_records(department_id, award_date, risk_score)
INCLUDE (tender_id, tender_title, awarded_amount);
```

**3. Read Replicas**:
- Route read queries to replicas
- Write queries to primary
- Use connection pooling (PgBouncer)

### Caching Strategy

**Multi-Level Caching**:

```
Level 1: Browser Cache (static assets)
         ↓
Level 2: CDN (images, CSS, JS)
         ↓
Level 3: Redis (API responses)
         ↓
Level 4: PostgreSQL Query Cache
         ↓
Level 5: Disk Storage
```

**Cache Invalidation**:
```python
def invalidate_related_caches(procurement_id):
    """
    Invalidate caches when data changes
    """
    # Get affected keys
    record = db.get_procurement(procurement_id)
    
    # Invalidate specific record cache
    cache.delete(f"procurement:record:{procurement_id}")
    
    # Invalidate department stats
    cache.delete_pattern(f"dashboard:stats:*{record.department_id}*")
    
    # Invalidate list caches (could be many)
    cache.delete_pattern("procurement:list:*")
```

### Asynchronous Processing

**Background Job Queue**:

```python
# Using Celery for async tasks
from celery import Celery

app = Celery('procurement_transparency')

@app.task
def analyze_procurement_batch(procurement_ids):
    """
    Analyze multiple procurement records asynchronously
    """
    for proc_id in procurement_ids:
        record = db.get_procurement(proc_id)
        analysis = risk_engine.analyze(record)
        db.store_analysis(analysis)

# Trigger from upload endpoint
@app.post('/api/upload')
def upload_csv(file):
    # ... validation and insertion ...
    
    # Queue risk analysis
    analyze_procurement_batch.delay(inserted_ids)
    
    return {"status": "processing"}
```

### Performance Targets

**Hackathon MVP**:
- Dataset: 10,000 records
- Dashboard load: < 3s (p95)
- Risk analysis: < 5s per record
- Concurrent users: 50

**Production Scale**:
- Dataset: 1,000,000+ records
- Dashboard load: < 2s (p95)
- Risk analysis: < 2s per record
- Concurrent users: 10,000+
- Uptime: 99.9%


## Security and Ethical Safeguards

### Authentication & Authorization

**JWT Implementation**:

```javascript
// Token structure
{
  "header": {
    "alg": "HS256",
    "typ": "JWT"
  },
  "payload": {
    "user_id": "uuid",
    "email": "user@example.com",
    "role": "citizen",
    "iat": 1708000000,
    "exp": 1708604800  // 7 days
  },
  "signature": "..."
}

// Middleware
function authenticateToken(req, res, next) {
    const token = req.headers['authorization']?.split(' ')[1];
    
    if (!token) {
        return res.status(401).json({ error: 'No token provided' });
    }
    
    jwt.verify(token, process.env.JWT_SECRET, (err, user) => {
        if (err) {
            return res.status(403).json({ error: 'Invalid token' });
        }
        req.user = user;
        next();
    });
}
```

### Data Security

**1. Encryption**:
```python
from cryptography.fernet import Fernet
import bcrypt

# Password hashing
def hash_password(password: str) -> str:
    salt = bcrypt.gensalt(rounds=12)
    return bcrypt.hashpw(password.encode(), salt).decode()

def verify_password(password: str, hash: str) -> bool:
    return bcrypt.checkpw(password.encode(), hash.encode())

# Sensitive data encryption
cipher = Fernet(os.environ['ENCRYPTION_KEY'])

def encrypt_email(email: str) -> str:
    return cipher.encrypt(email.encode()).decode()

def decrypt_email(encrypted: str) -> str:
    return cipher.decrypt(encrypted.encode()).decode()
```

**2. SQL Injection Prevention**:
```python
# Use parameterized queries
def get_procurement_by_department(department_id: str):
    query = """
        SELECT * FROM procurement_records 
        WHERE department_id = %s
    """
    # Safe: parameters passed separately
    cursor.execute(query, (department_id,))
    return cursor.fetchall()

# NEVER do this:
# query = f"SELECT * FROM procurement_records WHERE department_id = '{department_id}'"
```

**3. Rate Limiting**:
```javascript
const rateLimit = require('express-rate-limit');

// General API rate limit
const apiLimiter = rateLimit({
    windowMs: 15 * 60 * 1000, // 15 minutes
    max: 100, // 100 requests per window
    message: 'Too many requests, please try again later'
});

// Stricter limit for auth endpoints
const authLimiter = rateLimit({
    windowMs: 15 * 60 * 1000,
    max: 5, // 5 login attempts
    skipSuccessfulRequests: true
});

app.use('/api/', apiLimiter);
app.use('/api/auth/login', authLimiter);
```

### Ethical Safeguards Implementation

**1. Disclaimer Injection**:
```javascript
// Middleware to add disclaimers to all risk responses
function addEthicalDisclaimer(req, res, next) {
    const originalJson = res.json;
    
    res.json = function(data) {
        if (data.risk_score !== undefined) {
            data.disclaimer = "Risk scores are analytical indicators, not evidence of wrongdoing. Further investigation is required before drawing conclusions.";
        }
        originalJson.call(this, data);
    };
    
    next();
}

app.use('/api/procurement', addEthicalDisclaimer);
```

**2. Non-Accusatory Language Enforcement**:
```python
# Template validation
FORBIDDEN_PHRASES = [
    'proves corruption',
    'guilty of',
    'fraudulent',
    'criminal activity',
    'definitely corrupt'
]

def validate_explanation_text(text: str) -> bool:
    """
    Ensure explanation doesn't contain accusatory language
    """
    text_lower = text.lower()
    for phrase in FORBIDDEN_PHRASES:
        if phrase in text_lower:
            raise ValueError(f"Accusatory language detected: {phrase}")
    return True
```

**3. Vendor Anonymization (Optional)**:
```python
def anonymize_vendor_id(vendor_id: str, show_full: bool = False) -> str:
    """
    Partially anonymize vendor IDs for public display
    """
    if show_full:
        return vendor_id
    
    # Show first 4 and last 2 characters
    if len(vendor_id) > 6:
        return f"{vendor_id[:4]}***{vendor_id[-2:]}"
    return "***"
```

**4. Audit Logging**:
```python
def log_sensitive_action(user_id: str, action: str, details: dict):
    """
    Log all sensitive operations for accountability
    """
    audit_log = {
        'user_id': user_id,
        'action': action,
        'details': details,
        'ip_address': request.remote_addr,
        'user_agent': request.headers.get('User-Agent'),
        'timestamp': datetime.utcnow()
    }
    
    db.insert('audit_logs', audit_log)

# Usage
@app.post('/api/rti/generate')
def generate_rti():
    # ... RTI generation logic ...
    
    log_sensitive_action(
        user_id=current_user.id,
        action='RTI_GENERATED',
        details={'procurement_id': procurement_id}
    )
```

### HTTPS and Security Headers

```javascript
const helmet = require('helmet');

app.use(helmet({
    contentSecurityPolicy: {
        directives: {
            defaultSrc: ["'self'"],
            styleSrc: ["'self'", "'unsafe-inline'"],
            scriptSrc: ["'self'"],
            imgSrc: ["'self'", "data:", "https:"],
        }
    },
    hsts: {
        maxAge: 31536000,
        includeSubDomains: true,
        preload: true
    }
}));

// Force HTTPS in production
if (process.env.NODE_ENV === 'production') {
    app.use((req, res, next) => {
        if (req.header('x-forwarded-proto') !== 'https') {
            res.redirect(`https://${req.header('host')}${req.url}`);
        } else {
            next();
        }
    });
}
```

## Deployment Architecture

### Hackathon MVP Deployment

**Single-Server Setup (Heroku/Vercel)**:

```
┌─────────────────────────────────────────┐
│         Heroku Dyno / EC2 Instance      │
│                                         │
│  ┌─────────────────────────────────┐   │
│  │  Frontend (React Build)         │   │
│  │  Served by Express static       │   │
│  └─────────────────────────────────┘   │
│                                         │
│  ┌─────────────────────────────────┐   │
│  │  Backend API (Express/Flask)    │   │
│  │  Port: 3000 / 5000              │   │
│  └─────────────────────────────────┘   │
│                                         │
│  ┌─────────────────────────────────┐   │
│  │  Risk Engine (Python)           │   │
│  │  Called via subprocess/API      │   │
│  └─────────────────────────────────┘   │
└─────────────────────────────────────────┘
                  │
                  ▼
┌─────────────────────────────────────────┐
│  Heroku Postgres / RDS Free Tier        │
│  - 10,000 rows limit (Heroku free)      │
│  - 20GB storage (AWS free tier)         │
└─────────────────────────────────────────┘
```

**Environment Variables**:
```bash
# .env file
NODE_ENV=production
PORT=3000
DATABASE_URL=postgresql://user:pass@host:5432/dbname
JWT_SECRET=your-secret-key-here
ENCRYPTION_KEY=your-encryption-key-here
REDIS_URL=redis://localhost:6379
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_USER=your-email@gmail.com
SMTP_PASS=your-app-password
FRONTEND_URL=https://your-app.herokuapp.com
```

**Dockerfile** (for containerized deployment):

```dockerfile
# Multi-stage build
FROM node:18-alpine AS frontend-build
WORKDIR /app/frontend
COPY frontend/package*.json ./
RUN npm ci
COPY frontend/ ./
RUN npm run build

FROM python:3.9-slim AS backend
WORKDIR /app

# Install system dependencies
RUN apt-get update && apt-get install -y \
    postgresql-client \
    && rm -rf /var/lib/apt/lists/*

# Install Python dependencies
COPY requirements.txt ./
RUN pip install --no-cache-dir -r requirements.txt

# Install Node.js for API server
RUN apt-get update && apt-get install -y nodejs npm

# Copy backend code
COPY backend/ ./backend/
COPY risk_engine/ ./risk_engine/

# Copy frontend build
COPY --from=frontend-build /app/frontend/build ./backend/public

# Expose port
EXPOSE 3000

# Start command
CMD ["node", "backend/server.js"]
```

**docker-compose.yml** (for local development):

```yaml
version: '3.8'

services:
  postgres:
    image: postgres:14-alpine
    environment:
      POSTGRES_DB: procurement_transparency
      POSTGRES_USER: admin
      POSTGRES_PASSWORD: password
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"

  backend:
    build: .
    ports:
      - "3000:3000"
    environment:
      DATABASE_URL: postgresql://admin:password@postgres:5432/procurement_transparency
      REDIS_URL: redis://redis:6379
      JWT_SECRET: dev-secret-key
    depends_on:
      - postgres
      - redis
    volumes:
      - ./backend:/app/backend
      - ./risk_engine:/app/risk_engine

volumes:
  postgres_data:
```

### Production Deployment (AWS)

**Architecture**:

```
                    ┌─────────────────┐
                    │   CloudFront    │
                    │   (CDN + SSL)   │
                    └─────────────────┘
                            │
                            ▼
                    ┌─────────────────┐
                    │  Application    │
                    │  Load Balancer  │
                    └─────────────────┘
                            │
        ┌───────────────────┼───────────────────┐
        │                   │                   │
        ▼                   ▼                   ▼
┌───────────────┐   ┌───────────────┐   ┌───────────────┐
│  ECS Task 1   │   │  ECS Task 2   │   │  ECS Task N   │
│  (Container)  │   │  (Container)  │   │  (Container)  │
└───────────────┘   └───────────────┘   └───────────────┘
        │                   │                   │
        └───────────────────┼───────────────────┘
                            │
        ┌───────────────────┼───────────────────┐
        │                   │                   │
        ▼                   ▼                   ▼
┌───────────────┐   ┌───────────────┐   ┌───────────────┐
│  RDS Primary  │   │  ElastiCache  │   │      S3       │
│  (PostgreSQL) │   │    (Redis)    │   │  (File Store) │
└───────────────┘   └───────────────┘   └───────────────┘
        │
        ▼
┌───────────────┐
│  RDS Replica  │
│  (Read-only)  │
└───────────────┘
```

**Infrastructure as Code (Terraform)**:

```hcl
# main.tf
provider "aws" {
  region = "ap-south-1"  # Mumbai region for India deployment
}

# VPC and networking
resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"
  
  tags = {
    Name = "procurement-transparency-vpc"
  }
}

# RDS PostgreSQL
resource "aws_db_instance" "postgres" {
  identifier           = "procurement-db"
  engine              = "postgres"
  engine_version      = "14.7"
  instance_class      = "db.t3.micro"
  allocated_storage   = 20
  storage_encrypted   = true
  
  db_name  = "procurement_transparency"
  username = var.db_username
  password = var.db_password
  
  backup_retention_period = 7
  multi_az               = true
  
  tags = {
    Name = "procurement-transparency-db"
  }
}

# ElastiCache Redis
resource "aws_elasticache_cluster" "redis" {
  cluster_id           = "procurement-cache"
  engine              = "redis"
  node_type           = "cache.t3.micro"
  num_cache_nodes     = 1
  parameter_group_name = "default.redis7"
  port                = 6379
}

# ECS Cluster
resource "aws_ecs_cluster" "main" {
  name = "procurement-transparency-cluster"
}

# Application Load Balancer
resource "aws_lb" "main" {
  name               = "procurement-alb"
  internal           = false
  load_balancer_type = "application"
  security_groups    = [aws_security_group.alb.id]
  subnets           = aws_subnet.public[*].id
}
```

### CI/CD Pipeline (GitHub Actions)

```yaml
# .github/workflows/deploy.yml
name: Deploy to Production

on:
  push:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '18'
      
      - name: Setup Python
        uses: actions/setup-python@v4
        with:
          python-version: '3.9'
      
      - name: Install dependencies
        run: |
          cd frontend && npm ci
          cd ../backend && npm ci
          pip install -r requirements.txt
      
      - name: Run tests
        run: |
          cd frontend && npm test
          cd ../backend && npm test
          pytest risk_engine/tests/
      
      - name: Run linting
        run: |
          cd frontend && npm run lint
          cd ../backend && npm run lint
          flake8 risk_engine/

  build:
    needs: test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v2
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ap-south-1
      
      - name: Login to Amazon ECR
        id: login-ecr
        uses: aws-actions/amazon-ecr-login@v1
      
      - name: Build and push Docker image
        env:
          ECR_REGISTRY: ${{ steps.login-ecr.outputs.registry }}
          ECR_REPOSITORY: procurement-transparency
          IMAGE_TAG: ${{ github.sha }}
        run: |
          docker build -t $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG .
          docker push $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG
      
      - name: Deploy to ECS
        run: |
          aws ecs update-service \
            --cluster procurement-transparency-cluster \
            --service procurement-api \
            --force-new-deployment
```

### Monitoring and Observability

**Logging Strategy**:

```javascript
const winston = require('winston');

const logger = winston.createLogger({
    level: 'info',
    format: winston.format.json(),
    defaultMeta: { service: 'procurement-api' },
    transports: [
        new winston.transports.File({ filename: 'error.log', level: 'error' }),
        new winston.transports.File({ filename: 'combined.log' }),
    ],
});

// Log all API requests
app.use((req, res, next) => {
    logger.info('API Request', {
        method: req.method,
        path: req.path,
        ip: req.ip,
        user_id: req.user?.id
    });
    next();
});
```

**Health Check Endpoint**:

```javascript
app.get('/health', async (req, res) => {
    const health = {
        uptime: process.uptime(),
        timestamp: Date.now(),
        status: 'OK',
        checks: {}
    };
    
    // Database check
    try {
        await db.query('SELECT 1');
        health.checks.database = 'OK';
    } catch (err) {
        health.checks.database = 'FAIL';
        health.status = 'DEGRADED';
    }
    
    // Redis check
    try {
        await redis.ping();
        health.checks.redis = 'OK';
    } catch (err) {
        health.checks.redis = 'FAIL';
        health.status = 'DEGRADED';
    }
    
    const statusCode = health.status === 'OK' ? 200 : 503;
    res.status(statusCode).json(health);
});
```

**Metrics Collection** (Prometheus):

```javascript
const promClient = require('prom-client');

// Create metrics
const httpRequestDuration = new promClient.Histogram({
    name: 'http_request_duration_seconds',
    help: 'Duration of HTTP requests in seconds',
    labelNames: ['method', 'route', 'status_code']
});

const riskAnalysisCounter = new promClient.Counter({
    name: 'risk_analysis_total',
    help: 'Total number of risk analyses performed',
    labelNames: ['pattern_type']
});

// Middleware to track request duration
app.use((req, res, next) => {
    const start = Date.now();
    
    res.on('finish', () => {
        const duration = (Date.now() - start) / 1000;
        httpRequestDuration
            .labels(req.method, req.route?.path || req.path, res.statusCode)
            .observe(duration);
    });
    
    next();
});

// Metrics endpoint
app.get('/metrics', async (req, res) => {
    res.set('Content-Type', promClient.register.contentType);
    res.end(await promClient.register.metrics());
});
```

## Testing Strategy

### Unit Tests

**Frontend (Jest + React Testing Library)**:

```javascript
// components/__tests__/SummaryCards.test.jsx
import { render, screen } from '@testing-library/react';
import SummaryCards from '../SummaryCards';

describe('SummaryCards', () => {
    test('displays correct statistics', () => {
        const stats = {
            total_contracts: 1000,
            flagged_count: 150,
            flagged_percentage: 15.0,
            avg_risk_score: 45.2
        };
        
        render(<SummaryCards stats={stats} />);
        
        expect(screen.getByText('1,000')).toBeInTheDocument();
        expect(screen.getByText('150')).toBeInTheDocument();
        expect(screen.getByText('15.0%')).toBeInTheDocument();
    });
});
```

**Backend (Jest/Mocha)**:

```javascript
// controllers/__tests__/procurementController.test.js
const request = require('supertest');
const app = require('../app');

describe('GET /api/procurement', () => {
    test('returns paginated results', async () => {
        const response = await request(app)
            .get('/api/procurement?page=1&limit=50')
            .expect(200);
        
        expect(response.body).toHaveProperty('data');
        expect(response.body).toHaveProperty('pagination');
        expect(response.body.data).toHaveLength(50);
    });
    
    test('filters by department', async () => {
        const response = await request(app)
            .get('/api/procurement?department=DEPT001')
            .expect(200);
        
        response.body.data.forEach(record => {
            expect(record.department_id).toBe('DEPT001');
        });
    });
});
```

**Risk Engine (pytest)**:

```python
# risk_engine/tests/test_single_bidder.py
import pytest
from detectors.single_bidder import SingleBidderDetector
from models import ProcurementRecord

def test_single_bidder_detection():
    detector = SingleBidderDetector()
    
    record = ProcurementRecord(
        tender_id='T001',
        bidder_count=1,
        tender_value=5000000,
        category='IT_HARDWARE'
    )
    
    pattern = detector.detect(record)
    
    assert pattern is not None
    assert pattern.type == 'SINGLE_BIDDER'
    assert pattern.score >= 60
    assert pattern.score <= 95

def test_multiple_bidders_no_detection():
    detector = SingleBidderDetector()
    
    record = ProcurementRecord(
        tender_id='T002',
        bidder_count=5,
        tender_value=5000000,
        category='IT_HARDWARE'
    )
    
    pattern = detector.detect(record)
    
    assert pattern is None
```

### Integration Tests

```python
# tests/integration/test_risk_analysis_flow.py
import pytest
from app import create_app
from database import db

@pytest.fixture
def client():
    app = create_app('testing')
    with app.test_client() as client:
        with app.app_context():
            db.create_all()
            yield client
            db.drop_all()

def test_upload_and_analyze_flow(client):
    # Upload CSV
    with open('tests/fixtures/sample_procurement.csv', 'rb') as f:
        response = client.post('/api/upload', data={'file': f})
    
    assert response.status_code == 200
    data = response.get_json()
    assert data['successful_rows'] > 0
    
    # Query procurement records
    response = client.get('/api/procurement')
    assert response.status_code == 200
    records = response.get_json()['data']
    assert len(records) > 0
    
    # Check risk analysis exists
    procurement_id = records[0]['procurement_id']
    response = client.get(f'/api/procurement/{procurement_id}')
    assert response.status_code == 200
    detail = response.get_json()
    assert 'risk_analysis' in detail
```

### End-to-End Tests (Cypress)

```javascript
// cypress/e2e/dashboard.cy.js
describe('Dashboard Flow', () => {
    beforeEach(() => {
        cy.visit('/dashboard');
        cy.login('test@example.com', 'password123');
    });
    
    it('displays procurement records', () => {
        cy.get('[data-testid="procurement-table"]').should('be.visible');
        cy.get('[data-testid="procurement-row"]').should('have.length.at.least', 1);
    });
    
    it('filters by department', () => {
        cy.get('[data-testid="department-filter"]').select('DEPT001');
        cy.get('[data-testid="apply-filters"]').click();
        
        cy.get('[data-testid="procurement-row"]').each($row => {
            cy.wrap($row).should('contain', 'DEPT001');
        });
    });
    
    it('generates RTI request', () => {
        cy.get('[data-testid="procurement-row"]').first().click();
        cy.get('[data-testid="generate-rti"]').click();
        
        cy.get('[data-testid="applicant-name"]').type('Test User');
        cy.get('[data-testid="applicant-address"]').type('123 Test St');
        cy.get('[data-testid="applicant-email"]').type('test@example.com');
        cy.get('[data-testid="submit-rti"]').click();
        
        cy.get('[data-testid="download-rti"]').should('be.visible');
    });
});
```

## Performance Optimization

### Database Query Optimization

```sql
-- Before: Slow query (full table scan)
SELECT * FROM procurement_records 
WHERE department_id = 'DEPT001' 
  AND award_date >= '2025-01-01'
ORDER BY risk_score DESC;

-- After: Optimized with covering index
CREATE INDEX idx_dept_date_risk_covering 
ON procurement_records(department_id, award_date, risk_score)
INCLUDE (tender_id, tender_title, awarded_amount, awarded_vendor_name);

-- Query plan shows index-only scan
EXPLAIN ANALYZE SELECT * FROM procurement_records 
WHERE department_id = 'DEPT001' 
  AND award_date >= '2025-01-01'
ORDER BY risk_score DESC;
```

### Frontend Optimization

**Code Splitting**:

```javascript
// Lazy load routes
import { lazy, Suspense } from 'react';

const Dashboard = lazy(() => import('./components/Dashboard'));
const RTIGenerator = lazy(() => import('./components/RTI/RTIGenerator'));

function App() {
    return (
        <Suspense fallback={<LoadingSpinner />}>
            <Routes>
                <Route path="/dashboard" element={<Dashboard />} />
                <Route path="/rti/generate" element={<RTIGenerator />} />
            </Routes>
        </Suspense>
    );
}
```

**Memoization**:

```javascript
import { useMemo, useCallback } from 'react';

function ProcurementTable({ data, filters }) {
    // Memoize filtered data
    const filteredData = useMemo(() => {
        return data.filter(record => {
            if (filters.department && record.department_id !== filters.department) {
                return false;
            }
            if (filters.risk_level && record.risk_level !== filters.risk_level) {
                return false;
            }
            return true;
        });
    }, [data, filters]);
    
    // Memoize callback
    const handleRowClick = useCallback((record) => {
        navigate(`/procurement/${record.procurement_id}`);
    }, [navigate]);
    
    return (
        <Table data={filteredData} onRowClick={handleRowClick} />
    );
}
```

### API Response Compression

```javascript
const compression = require('compression');

app.use(compression({
    filter: (req, res) => {
        if (req.headers['x-no-compression']) {
            return false;
        }
        return compression.filter(req, res);
    },
    level: 6
}));
```

## Conclusion

This design document provides a comprehensive technical blueprint for the Procurement Transparency AI system. The architecture balances hackathon feasibility with production scalability, emphasizing:

1. **Explainability**: Every risk detection is backed by clear evidence and plain-language explanations
2. **Ethical Design**: Built-in safeguards prevent misuse and ensure non-accusatory framing
3. **Scalability**: Modular architecture supports growth from 10K to 1M+ records
4. **Citizen-Centric**: Intuitive UI and RTI generation empower non-technical users
5. **Technical Excellence**: Modern stack, best practices, comprehensive testing

The system serves as a transparency enhancement layer, not an enforcement mechanism, empowering citizens to monitor public procurement through data-driven insights while maintaining ethical boundaries.

