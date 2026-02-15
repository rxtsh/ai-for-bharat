# Requirements Document: Procurement Transparency AI

## Introduction

This document specifies requirements for an AI-powered transparency enhancement layer for public procurement monitoring in India. The system analyzes structured procurement data from government e-procurement portals (GEM, state tender portals) to detect statistical anomalies and risk patterns using rule-based heuristics and machine learning models. It enables citizens and civil society organizations to generate legally compliant RTI requests and monitor procurement through an explainable, evidence-based dashboard.

**Technical Scope**: The system processes structured procurement datasets (CSV/JSON), applies statistical analysis and NLP-based pattern detection, and presents findings through a web-based dashboard with RESTful API backend. The MVP focuses on demonstrable risk detection algorithms with clear explainability, not comprehensive national deployment.

**Ethical Foundation**: All outputs are framed as risk indicators requiring investigation, never as accusations. The system serves as a transparency tool, not an enforcement mechanism.

## Glossary

- **System**: The Procurement Transparency AI platform (web application with backend API)
- **Risk_Detector**: Node.js-based analysis engine using statistical libraries and LLM API for pattern detection
- **RTI_Generator**: Template engine producing RTI Act 2005 compliant documents in structured format
- **Dashboard**: Next.js-based web interface with data visualization using D3.js or Chart.js
- **Citizen_User**: Non-technical end user with basic web browsing skills
- **Procurement_Record**: Structured data containing tender_id, department, vendor, amount, dates, specifications
- **Risk_Score**: Weighted numerical indicator (0-100) computed from multiple risk pattern detections
- **Risk_Pattern**: Enumerated anomaly type (SINGLE_BIDDER, VENDOR_REPETITION, COMPRESSED_DEADLINE, BUDGET_ANOMALY, SPEC_TAILORING)
- **Explainability_Report**: JSON structure containing risk factors, scores, evidence citations, and plain-language summary
- **RTI_Request**: Text document following Section 6 format of RTI Act 2005 with applicant details, information sought, and fee payment reference
- **Vendor**: Entity identified by PAN number or unique vendor ID in procurement systems
- **Tender**: Procurement opportunity with fields: tender_id, title, department, estimated_value, deadline, specifications
- **Contract_Award**: Record linking tender_id to winning vendor_id with awarded_amount and award_date
- **Specification_Tailoring**: NLP-detected patterns including brand names, proprietary terms, or overly restrictive technical requirements
- **Budget_Anomaly**: Statistical outlier where awarded_amount deviates >2 standard deviations from category mean
- **GEM_Portal**: Government e-Marketplace (gem.gov.in) - primary data source for central government procurement
- **Historical_Baseline**: Statistical distribution of contract values, timelines, and vendor patterns computed from past 24 months of data

## Technical Architecture Overview

### System Components

**Frontend**: React single-page application with responsive design
- UI Framework: React 18+ with functional components and hooks
- Visualization: Chart.js for time series, D3.js for network graphs, Leaflet for maps
- State Management: React Context API or Redux for global state
- Styling: Tailwind CSS or Bootstrap for responsive layout

**Backend**: RESTful API server
- Framework: Express.js (Node.js)
- API Design: REST with JSON payloads, JWT authentication
- Endpoints: /api/procurement, /api/auth, /api/upload, /api/rti

**Risk Detection Engine**: Node.js-based analysis module
- Statistical Analysis: Math.js, simple-statistics for data processing and outlier detection
- NLP: LLM API (OpenAI GPT-4 or Anthropic Claude) for specification text analysis
- Pattern Detection: Rule-based heuristics with configurable thresholds
- Explainability: Template-based natural language generation using LLM

**Database**: MongoDB for flexible document storage
- Collections: procurement_records, users, risk_flags, explainability_reports, audit_logs
- Indexes: (department_id, award_date), (vendor_id, award_date), (risk_score)
- Constraints: Unique compound indexes on (tender_id, department_id)

**Caching Layer**: Redis (optional for production)
- Cache dashboard queries with 5-minute TTL
- Session storage for JWT tokens
- Rate limiting counters

### Data Flow

1. **Ingestion**: CSV upload → Schema validation → Normalization → Database insert
2. **Analysis**: Procurement record → Risk detector → Pattern checks → Score calculation → Explainability generation
3. **Visualization**: User request → API query → Database fetch → JSON response → Frontend render
4. **RTI Generation**: User selection → Template engine → Field population → PDF generation → Download

### Deployment Architecture (Hackathon MVP)

- **Hosting**: Single server deployment (Heroku, Vercel, or AWS EC2 t2.micro)
- **Database**: MongoDB Atlas (free tier M0 cluster with 512MB storage)
- **Storage**: Local filesystem for uploaded CSVs (production would use S3)
- **CI/CD**: GitHub Actions for automated testing and deployment

## Requirements

### Requirement 1: Single Bidder Detection

**User Story:** As a citizen user, I want to identify tenders with only one bidder, so that I can flag potential competition suppression.

**Implementation Context**: Analyze the bidder_count field in procurement records. For hackathon MVP, use simple threshold-based detection with value-weighted scoring.

#### Acceptance Criteria

1. WHEN a procurement record contains bidder_count equal to 1, THE Risk_Detector SHALL flag it with risk_pattern SINGLE_BIDDER
2. WHEN calculating risk score for SINGLE_BIDDER pattern, THE Risk_Detector SHALL apply formula: base_score = 60 + min(20, tender_value_lakhs / 50)
3. WHEN displaying SINGLE_BIDDER flags, THE System SHALL generate explainability_report containing: tender_id, department, tender_value, expected_bidder_count (from category average), and plain-language explanation
4. WHEN tender_value exceeds 1000000 rupees AND bidder_count equals 1, THE Risk_Detector SHALL add 15 points to risk_score
5. THE Dashboard SHALL compute and display single_bidder_percentage = (single_bidder_tenders / total_tenders) * 100 grouped by department

### Requirement 2: Repeated Vendor Win Detection

**User Story:** As a citizen user, I want to detect vendors winning multiple contracts from the same department, so that I can identify potential favoritism patterns.

**Implementation Context**: Query contract_awards table grouped by (vendor_id, department_id) with temporal windowing. Use SQL aggregation or pandas groupby for efficiency.

#### Acceptance Criteria

1. WHEN a vendor_id has contract_count > 3 from the same department_id within a 365-day rolling window, THE Risk_Detector SHALL flag it with risk_pattern VENDOR_REPETITION
2. WHEN calculating VENDOR_REPETITION risk, THE Risk_Detector SHALL compute: risk_score = min(100, 40 + (contract_count * 10) + (total_value_crores * 5))
3. WHEN total_value of contracts from one vendor to one department exceeds 10000000 rupees within 180 days, THE Risk_Detector SHALL assign risk_score >= 70
4. THE Explainability_Report SHALL include JSON array of contracts: [{contract_id, award_date, amount, tender_title}] sorted by award_date descending
5. THE Dashboard SHALL render network graph with nodes (departments, vendors) and edges (contract awards) where edge_thickness = log(contract_value) and node_color = risk_score_gradient

### Requirement 3: Compressed Deadline Detection

**User Story:** As a citizen user, I want to identify tenders with unrealistically short bidding windows, so that I can flag potential competition barriers.

**Implementation Context**: Calculate bidding_window = (submission_deadline - publication_date) in days. Compare against category-specific thresholds stored in configuration.

#### Acceptance Criteria

1. WHEN tender_value < 5000000 rupees AND bidding_window_days < 7, THE Risk_Detector SHALL flag it with risk_pattern COMPRESSED_DEADLINE
2. WHEN tender_value >= 5000000 rupees AND bidding_window_days < 14, THE Risk_Detector SHALL flag it with risk_pattern COMPRESSED_DEADLINE
3. WHEN calculating COMPRESSED_DEADLINE risk, THE Risk_Detector SHALL compute: risk_score = 50 + ((expected_days - actual_days) / expected_days) * 50, where expected_days = category_median from Historical_Baseline
4. THE Explainability_Report SHALL contain: actual_deadline_days, expected_deadline_days (from category median), deviation_percentage = ((expected - actual) / expected) * 100
5. WHEN COMPRESSED_DEADLINE co-occurs with SINGLE_BIDDER on same tender, THE Risk_Detector SHALL apply multiplicative factor: combined_risk_score = min(100, base_score * 1.15)

### Requirement 4: Budget Anomaly Detection

**User Story:** As a citizen user, I want to detect cost inflation and budget anomalies, so that I can identify potential financial irregularities.

**Implementation Context**: Use statistical outlier detection on awarded_amount compared to Historical_Baseline distribution for same procurement_category and geographic_region. Implement using scipy.stats or numpy percentile functions.

#### Acceptance Criteria

1. WHEN awarded_amount > (estimated_budget * 1.20), THE Risk_Detector SHALL flag it with risk_pattern BUDGET_ANOMALY
2. WHEN comparing contract prices, THE Risk_Detector SHALL query Historical_Baseline filtered by: procurement_category = current_category AND geographic_region = current_region AND award_year >= (current_year - 2)
3. WHEN awarded_amount > (category_mean + 2 * category_stddev), THE Risk_Detector SHALL assign risk_score >= 65 using formula: risk_score = 65 + min(35, z_score * 10) where z_score = (awarded_amount - mean) / stddev
4. THE Explainability_Report SHALL contain: awarded_amount, estimated_budget, historical_mean, historical_stddev, z_score, deviation_percentage = ((awarded - estimated) / estimated) * 100, sample_size (number of historical contracts used)
5. THE System SHALL maintain procurement_history table with schema: (contract_id, category, region, year, amount) indexed on (category, region, year) for query performance

### Requirement 5: Specification Tailoring Detection

**User Story:** As a citizen user, I want to identify tender specifications that may favor specific vendors, so that I can flag potential bid rigging.

**Implementation Context**: Apply NLP pattern matching on tender_specifications text field using regex for brand names and spaCy/NLTK for restrictive language detection. For hackathon MVP, use rule-based keyword matching with predefined tailoring indicator lists.

#### Acceptance Criteria

1. WHEN tender_specifications contains brand names from predefined_brand_list (e.g., "Dell", "HP", "Cisco", "Oracle"), THE Risk_Detector SHALL flag it with risk_pattern SPEC_TAILORING and sub_type BRAND_REFERENCE
2. WHEN tender_specifications contains phrases matching restrictive_pattern_regex (e.g., "only", "must be", "exclusively", "proprietary"), THE Risk_Detector SHALL increment tailoring_indicator_count
3. WHEN tailoring_indicator_count >= 3 OR brand_name_count >= 2, THE Risk_Detector SHALL assign risk_score = 70 + min(30, tailoring_indicator_count * 5)
4. THE Explainability_Report SHALL highlight matched_phrases as JSON array: [{phrase: "Dell servers only", position: char_offset, indicator_type: "BRAND_REFERENCE"}] and provide plain-language summary: "Specification contains 2 brand references and 3 restrictive phrases that may limit competition"
5. THE System SHALL maintain tailoring_knowledge_base as JSON configuration file containing: {brand_names: [], restrictive_patterns: [], exempted_categories: []} updatable without code changes

### Requirement 6: RTI Request Generation

**User Story:** As a citizen user, I want to generate legally sound RTI requests based on detected risks, so that I can obtain detailed information from government departments.

**Implementation Context**: Use Jinja2 or similar templating engine with predefined RTI templates. Populate fields from procurement_record and detected risk_patterns. Output as formatted text and PDF using ReportLab or WeasyPrint.

#### Acceptance Criteria

1. WHEN a citizen user clicks "Generate RTI" on a flagged procurement record, THE RTI_Generator SHALL create structured request using template: "To: Public Information Officer, [department_name]. Subject: Information regarding Tender [tender_id]. Under Section 6(1) of RTI Act 2005, I request the following information: [auto_generated_questions]"
2. THE RTI_Generator SHALL populate request fields: applicant_name (user input), applicant_address (user input), department_name (from procurement_record), tender_id, tender_title, risk_patterns_detected (comma-separated list)
3. WHEN generating RTI requests, THE System SHALL validate required fields: department_name NOT NULL, tender_id NOT NULL, applicant_name length >= 3, applicant_address length >= 10
4. THE RTI_Generator SHALL generate context-aware questions based on risk_pattern: IF SINGLE_BIDDER THEN add "1. How many vendors downloaded tender documents? 2. Were any vendors disqualified during technical evaluation?"; IF BUDGET_ANOMALY THEN add "1. Provide detailed cost breakdown. 2. Justification for amount exceeding estimate."
5. WHEN user clicks "Download RTI", THE System SHALL generate PDF with: A4 page size, Times New Roman 12pt font, proper spacing per RTI format guidelines, placeholder for signature, and footer text: "Generated by Procurement Transparency AI - verify details before submission"
6. THE System SHALL provide downloadable output in both TXT format (for copy-paste to online RTI portals) and PDF format (for postal submission)

### Requirement 7: Explainable Risk Scoring

**User Story:** As a citizen user, I want to understand why a procurement record is flagged as risky, so that I can make informed decisions about follow-up actions.

**Implementation Context**: Generate explainability_report as structured JSON with human-readable summary. Use template-based natural language generation for plain-language explanations.

#### Acceptance Criteria

1. WHEN the Risk_Detector assigns risk_score, THE System SHALL generate explainability_report with schema: {tender_id, overall_risk_score, risk_patterns: [{pattern_type, score_contribution, evidence, explanation}], summary_text, disclaimer}
2. THE explainability_report.risk_patterns array SHALL list all detected patterns with individual score_contribution values summing to overall_risk_score (weighted combination, not simple addition)
3. WHEN displaying risk_score on Dashboard, THE System SHALL apply color_code: IF score < 40 THEN "green", IF 40 <= score <= 70 THEN "yellow", IF score > 70 THEN "red", with legend: "Green: Low Risk, Yellow: Medium Risk, Red: High Risk - Further Investigation Recommended"
4. THE explainability_report.summary_text SHALL use non-accusatory language template: "This tender shows [count] risk indicators: [pattern_list]. These patterns suggest potential issues requiring further investigation, not proof of wrongdoing."
5. WHEN multiple risk_patterns are detected, THE System SHALL compute combined_risk_score using weighted formula: combined_score = min(100, sum(pattern_scores * pattern_weights) * interaction_multiplier) where interaction_multiplier = 1.0 + (0.05 * (pattern_count - 1))
6. THE System SHALL display educational tooltips on hover: SINGLE_BIDDER → "Tenders with only one bidder may indicate limited competition or barriers to entry", BUDGET_ANOMALY → "Significant cost deviations from historical averages may warrant scrutiny"

### Requirement 8: Public Monitoring Dashboard

**User Story:** As a citizen user, I want to visualize procurement data and risk patterns across departments and time periods, so that I can identify systemic issues.

**Implementation Context**: Build React SPA with RESTful API backend. Use Chart.js for time series, D3.js for network graphs, and Leaflet for geographic maps. Implement server-side filtering and pagination for performance.

#### Acceptance Criteria

1. THE Dashboard SHALL display procurement records in paginated table (50 records per page) with filters: department_dropdown, date_range_picker (start_date, end_date), value_range_slider (min_amount, max_amount), risk_score_filter (checkboxes: Low/Medium/High)
2. WHEN a citizen user loads dashboard, THE System SHALL display summary_statistics card: total_contracts_count, flagged_contracts_count, flagged_percentage = (flagged / total) * 100, average_risk_score = mean(risk_scores), highest_risk_department
3. THE Dashboard SHALL render time_series_chart showing risk_pattern_frequency over time with: x-axis = month buckets, y-axis = count of flagged contracts, separate lines for each risk_pattern_type, interactive tooltips on hover
4. WHEN user clicks table row for flagged procurement_record, THE Dashboard SHALL open detail_modal displaying: full procurement details, explainability_report (formatted JSON), risk_score_breakdown (pie chart of score contributions), "Generate RTI" button
5. THE Dashboard SHALL provide choropleth_map visualization with: India state boundaries, color intensity = average_risk_score per state, click-to-drill-down to district level, legend showing score ranges
6. THE System SHALL fetch dashboard data via API endpoint GET /api/procurement?filters={} with response_time < 2 seconds for datasets up to 10,000 records, using database indexes on (department_id, award_date, risk_score)

### Requirement 9: Data Ingestion and Processing

**User Story:** As a system administrator, I want to ingest procurement data from multiple government sources, so that the system has comprehensive coverage.

**Implementation Context**: Build ETL pipeline using Node.js scripts. For hackathon MVP, support CSV upload with schema validation. Production would add API connectors for GEM portal and state e-procurement systems.

#### Acceptance Criteria

1. THE System SHALL ingest procurement data via POST /api/upload endpoint accepting CSV files with required columns: tender_id, department, tender_title, estimated_budget, publication_date, submission_deadline, bidder_count, awarded_vendor_id, awarded_amount, award_date, specifications_text
2. WHEN CSV upload is received, THE System SHALL validate schema using pandas DataFrame with dtype checks: tender_id (string), estimated_budget (float), dates (datetime), bidder_count (int), and return HTTP 400 with error_details JSON if validation fails
3. THE System SHALL normalize data using transformations: department_name → lowercase and trim whitespace, amounts → convert to rupees (handle lakhs/crores notation), dates → ISO 8601 format (YYYY-MM-DD)
4. WHEN ingesting records, THE System SHALL deduplicate using composite key (tender_id, department_id) with conflict resolution: keep record with latest updated_timestamp
5. THE System SHALL validate data completeness: IF tender_id IS NULL OR department IS NULL OR estimated_budget IS NULL THEN flag record with validation_status = "INCOMPLETE" and store in quarantine_table for manual review
6. WHEN data ingestion fails, THE System SHALL log errors to ingestion_log table with schema: (timestamp, filename, row_number, error_type, error_message) and return summary: {total_rows, successful_rows, failed_rows, error_log_id}

### Requirement 10: User Authentication and Access Control

**User Story:** As a citizen user, I want to create an account and save my monitoring preferences, so that I can track specific departments or risk patterns over time.

**Implementation Context**: Implement JWT-based authentication with bcrypt password hashing. Use MongoDB for user data storage. For hackathon MVP, basic email/password auth is sufficient; production would add OAuth.

#### Acceptance Criteria

1. THE System SHALL support user registration via POST /api/auth/register with payload: {email, password, full_name} and validation: email matches RFC 5322 regex, password length >= 8 characters with at least 1 uppercase, 1 lowercase, 1 digit
2. WHEN user registers, THE System SHALL send verification email to provided address with verification_token (UUID v4) valid for 24 hours, and set user.email_verified = false until token is confirmed via GET /api/auth/verify?token={verification_token}
3. THE System SHALL store user preferences in user_preferences table with schema: (user_id, watched_departments: JSON array, risk_threshold: int, email_alerts_enabled: boolean) and allow updates via PUT /api/user/preferences
4. WHEN user logs in via POST /api/auth/login, THE System SHALL validate credentials using bcrypt.compare(input_password, stored_hash) and return JWT token with payload: {user_id, email, exp: 7 days} signed with HS256 algorithm
5. THE System SHALL implement password reset flow: POST /api/auth/reset-request generates reset_token valid for 1 hour, sends email with reset link, POST /api/auth/reset-password validates token and updates password hash
6. WHEN user.last_login_date < (current_date - 180 days), THE System SHALL set user.account_status = "INACTIVE" and require reactivation via email verification before next login

### Requirement 11: Performance and Scalability

**User Story:** As a system administrator, I want the system to handle national-scale data volumes, so that it can serve users across India.

**Implementation Context**: Use MongoDB with proper indexing, Redis for caching, and horizontal scaling via containerization (Docker + Kubernetes for production). For hackathon, demonstrate performance with 10K record dataset.

#### Acceptance Criteria

1. THE System SHALL process CSV uploads of 10,000 procurement records within 600 seconds (average 60ms per record) using batch insert operations with chunk_size = 1000
2. WHEN Dashboard API endpoint GET /api/procurement receives 100 concurrent requests, THE System SHALL maintain p95 response_time < 3000ms using database connection pooling (pool_size = 20) and query result caching (TTL = 300 seconds)
3. THE System SHALL implement horizontal scaling architecture: stateless API servers behind load balancer, shared MongoDB database with replica sets, Redis cache cluster
4. WHEN Risk_Detector analyzes single procurement_record, THE System SHALL complete all pattern checks (5 risk patterns) within 5000ms using parallel execution where possible
5. THE System SHALL achieve 99.5% uptime during business hours (09:00-18:00 IST) measured as: (total_minutes - downtime_minutes) / total_minutes >= 0.995 over 30-day rolling window

### Requirement 12: Data Privacy and Security

**User Story:** As a citizen user, I want my personal information and monitoring activities to be protected, so that I can use the system safely.

**Implementation Context**: Implement security best practices using industry-standard libraries. Use Mongoose with built-in sanitization to prevent NoSQL injection, helmet.js for HTTP headers, and rate limiting via express-rate-limit.

#### Acceptance Criteria

1. THE System SHALL encrypt user passwords using bcrypt with cost_factor = 12 before storing in database, and encrypt sensitive user data (email, name) at rest using AES-256-GCM with key stored in environment variable
2. THE System SHALL enforce HTTPS for all API endpoints using TLS 1.3 with valid SSL certificate, and set security headers: Strict-Transport-Security, X-Content-Type-Options, X-Frame-Options
3. THE System SHALL not store user search queries or dashboard filter selections beyond 90 days, implementing automated cleanup job: DELETE FROM user_activity_log WHERE timestamp < (current_date - 90)
4. WHEN user requests account deletion via DELETE /api/user/account, THE System SHALL permanently remove user record and associated data within 30 days, logging deletion in audit_log with anonymized user_id
5. THE System SHALL implement rate limiting: 100 requests per 15 minutes per IP address for API endpoints, 5 login attempts per 15 minutes per email address, returning HTTP 429 when exceeded
6. THE System SHALL conduct security vulnerability scanning using OWASP ZAP or similar tool, and apply patches for CVSS score >= 7.0 vulnerabilities within 48 hours of disclosure

### Requirement 13: Ethical Safeguards

**User Story:** As a system administrator, I want to ensure the system does not enable harassment or false accusations, so that it serves its transparency mission responsibly.

**Implementation Context**: Implement content moderation, disclaimer text in UI templates, and audit logging. Use database triggers or application-level hooks to track algorithm changes.

#### Acceptance Criteria

1. THE System SHALL display disclaimer text on every risk_score display: "Risk scores are analytical indicators, not evidence of wrongdoing. Further investigation is required before drawing conclusions." with font_size >= 12px and prominent placement
2. THE System SHALL frame all explainability_report text using non-accusatory templates: "This pattern suggests..." instead of "This proves...", "May warrant investigation" instead of "Indicates corruption"
3. THE System SHALL not provide public comment or rating features on procurement records, vendors, or officials, limiting user interaction to: view data, generate RTI, save preferences
4. WHEN displaying vendor information, THE System SHALL show only: vendor_id (anonymized if PAN), company_name, contract_count, total_contract_value, without personal details like owner names, addresses, or contact information
5. THE System SHALL implement feedback mechanism via POST /api/feedback endpoint allowing departments to submit: {procurement_record_id, clarification_text, supporting_documents_urls} displayed alongside risk flags with label "Department Response"
6. THE System SHALL maintain algorithm_audit_log table with schema: (timestamp, change_description, changed_by, risk_pattern_affected, version_number) and display current algorithm version on Dashboard footer

### Requirement 14: Mobile Accessibility

**User Story:** As a citizen user, I want to access the system on my mobile device, so that I can monitor procurement on the go.

**Implementation Context**: Use responsive CSS framework (Bootstrap or Tailwind) with mobile-first design. Test on Chrome DevTools device emulation for common Android devices.

#### Acceptance Criteria

1. THE Dashboard SHALL implement responsive layout using CSS media queries with breakpoints: mobile (320px-767px), tablet (768px-1023px), desktop (1024px+), adapting grid columns and font sizes accordingly
2. WHEN accessed on mobile viewport_width < 768px, THE System SHALL: stack filters vertically, convert data tables to card layout, resize charts to fit screen width, increase touch target sizes to minimum 44x44px
3. THE System SHALL achieve Lighthouse mobile performance score >= 70 with metrics: First Contentful Paint < 3s, Time to Interactive < 5s, tested on simulated 3G network (750ms RTT, 1.6Mbps throughput)
4. THE RTI_Generator SHALL produce PDF files with mobile-optimized layout: single column, font_size >= 10pt, proper page breaks, viewable in mobile PDF readers without horizontal scrolling
5. THE System SHALL implement Progressive Web App features: manifest.json with app icons, service worker for offline caching of static assets, "Add to Home Screen" prompt after 2 visits

### Requirement 15: Multilingual Support

**User Story:** As a citizen user, I want to use the system in my preferred Indian language, so that language is not a barrier to transparency.

**Implementation Context**: Use i18n library (next-i18next for frontend, i18next for backend). Store translations in JSON files. For hackathon MVP, support English + Hindi; production would add more languages.

#### Acceptance Criteria

1. THE System SHALL support UI language selection via dropdown with options: English, Hindi, and minimum 3 additional languages (Tamil, Bengali, Marathi) with language codes: en, hi, ta, bn, mr
2. WHEN user selects language via PUT /api/user/preferences {language: "hi"}, THE System SHALL translate all UI elements (labels, buttons, help text, navigation) using translation_key lookup from language_files/{lang}.json
3. THE Explainability_Report SHALL generate summary_text in user's selected language using templated translations: "यह निविदा [count] जोखिम संकेतक दिखाती है" for Hindi, maintaining placeholder substitution for dynamic values
4. THE RTI_Generator SHALL produce requests in English (mandatory per RTI Act Section 6) with optional regional_language_summary section appended as: "सारांश (हिंदी में): [translated summary]"
5. THE System SHALL persist user language preference in user_preferences.language column and apply automatically on subsequent logins, defaulting to browser Accept-Language header for unauthenticated users

## Assumptions and Constraints

### Assumptions

1. Government procurement data is available through public sources (GEM portal downloads, state e-procurement portals, RTI responses) in structured formats (CSV, JSON, or scrapable HTML)
2. Users have basic digital literacy (can navigate websites, fill forms) and internet access via mobile or desktop
3. Historical procurement data for minimum 24 months is available to establish statistical baselines for anomaly detection
4. Government departments will not actively block system access to publicly available data (though API rate limits may apply)
5. Sample dataset of 5,000-10,000 procurement records is sufficient for hackathon demonstration and algorithm validation
6. Judges have technical background to evaluate AI methodology, code quality, and system architecture

### Constraints

1. **Legal Compliance**: System must comply with Right to Information Act 2005, IT Act 2000, and data protection regulations; cannot make legal determinations or accusations
2. **Hackathon Timeline**: MVP development constrained to 24-48 hours; must prioritize core risk detection and dashboard over advanced features
3. **Budget Limitations**: Must use open-source technologies (Node.js, MongoDB, Next.js) and free-tier cloud services (Heroku, Vercel, MongoDB Atlas, or AWS Free Tier); LLM API costs should be minimized through prompt optimization
4. **AI Explainability**: All risk detection algorithms must be interpretable and auditable; no black-box deep learning models without clear feature attribution
5. **Ethical Boundaries**: System cannot enable direct accusations, public shaming, or harassment; must maintain neutral, investigative framing
6. **Data Availability**: Limited to publicly available procurement data; cannot access confidential government databases or internal communications
7. **Technical Stack**: Must use widely-adopted technologies for maintainability and judge familiarity; avoid obscure frameworks or languages
8. **Performance**: Hackathon demo environment may have limited resources; system must function on single server with <4GB RAM for demonstration

## Risk and Mitigation Considerations

### Technical Risks

1. **Risk**: Inconsistent data formats across government portals
   **Mitigation**: Implement robust data normalization layer with manual review queue

2. **Risk**: False positive risk flags damaging legitimate vendors
   **Mitigation**: Conservative risk thresholds, clear disclaimers, and vendor feedback mechanism

3. **Risk**: System misuse for political targeting or harassment
   **Mitigation**: Ethical safeguards, usage monitoring, and terms of service enforcement

### Operational Risks

1. **Risk**: Government resistance or legal challenges
   **Mitigation**: Strict compliance with public data laws, transparency in methodology

2. **Risk**: Insufficient historical data for accurate anomaly detection
   **Mitigation**: Gradual model improvement as data accumulates, manual expert validation

## Success Metrics

### Hackathon Evaluation KPIs

1. **Functional Completeness** (Weight: 25%)
   - Metric: Percentage of core requirements implemented (Requirements 1-8 are core)
   - Target: >= 70% for viable MVP (minimum 5 of 8 core requirements)
   - Measurement: Feature checklist with demo validation

2. **Risk Detection Accuracy** (Weight: 20%)
   - Metric: Precision and recall on labeled test dataset of known high-risk cases
   - Target: Precision >= 75% (minimize false positives), Recall >= 60% (catch real issues)
   - Measurement: Confusion matrix on 100-record validation set with manual risk labels

3. **User Experience** (Weight: 15%)
   - Metric: Task completion rate for key user journey: "Find high-risk tender → Generate RTI request"
   - Target: >= 90% completion rate in user testing with 5 non-technical testers
   - Measurement: Timed usability test with success/failure tracking

4. **Performance** (Weight: 15%)
   - Metric: Dashboard load time (p95) and risk analysis processing time per record
   - Target: Dashboard < 3s, Risk analysis < 5s per record on 10K dataset
   - Measurement: Chrome DevTools Performance tab, backend timing logs

5. **Explainability** (Weight: 10%)
   - Metric: User comprehension score for risk explanations
   - Target: >= 80% of test users can correctly explain why a tender was flagged
   - Measurement: Post-demo questionnaire with comprehension questions

6. **Code Quality** (Weight: 10%)
   - Metric: Code organization, documentation, test coverage
   - Target: Modular architecture, README with setup instructions, >= 50% test coverage for core logic
   - Measurement: Judge code review, pytest/jest coverage reports

7. **Innovation** (Weight: 5%)
   - Metric: Novel application of AI/ML techniques, creative problem-solving
   - Target: Demonstrates unique approach beyond basic rule matching
   - Measurement: Qualitative judge assessment

### Long-term Impact Metrics (Post-Hackathon)

1. **Adoption Metrics**
   - Monthly active users (MAU) across Indian states
   - Number of RTI requests generated and submitted via platform
   - Geographic coverage (number of states/districts with active monitoring)

2. **Effectiveness Metrics**
   - Percentage of flagged cases resulting in government clarifications or investigations
   - False positive rate reduction over time as algorithms improve
   - Media citations and civil society organization partnerships

3. **System Health Metrics**
   - API uptime percentage (target: 99.5%)
   - Average response time under load
   - Data freshness (lag between procurement publication and system ingestion)

4. **Social Impact Metrics**
   - Government policy changes influenced by system insights
   - Reduction in single-bidder tender percentage in monitored departments
   - Citizen empowerment indicators (surveys, testimonials)

## Future Scope

### Phase 2 Enhancements (Post-Hackathon)

1. **Advanced Pattern Detection**
   - Graph neural networks for detecting collusion networks between vendors and officials
   - Time-series forecasting to predict high-risk procurement periods
   - Anomaly detection using isolation forests and autoencoders for complex patterns

2. **Predictive Analytics**
   - Risk forecasting: predict which departments/categories likely to have issues in next quarter
   - Budget optimization recommendations based on historical efficiency analysis
   - Vendor reputation scoring using longitudinal performance data

3. **Crowdsourced Validation**
   - Community verification system where users can validate or dispute risk flags
   - Reputation system for contributors with verified track records
   - Integration with investigative journalism platforms for collaborative research

4. **Legal Integration**
   - Direct connection to pro-bono legal aid networks for high-risk cases
   - Automated case file generation with evidence compilation
   - Integration with court case tracking systems to monitor outcomes

5. **Blockchain Audit Trail**
   - Immutable logging of all risk assessments and algorithm changes
   - Cryptographic verification of data provenance and analysis integrity
   - Public verifiability without compromising user privacy

6. **Automated RTI Filing**
   - Direct electronic submission to government RTI portals via API integration
   - Automated tracking of RTI response deadlines and appeals
   - Response parsing and structured data extraction from government replies

7. **Comparative Analysis**
   - Cross-state benchmarking of procurement efficiency and transparency
   - International comparison with procurement systems in other democracies
   - Best practice identification and dissemination

8. **Whistleblower Protection**
   - Secure anonymous tip submission with end-to-end encryption
   - Integration with secure communication channels (Signal, Tor)
   - Legal protection resources and guidance for whistleblowers

9. **Impact Tracking**
   - Longitudinal analysis of system effectiveness in reducing corruption indicators
   - A/B testing of different risk detection algorithms
   - Academic research partnerships for rigorous evaluation

10. **Public API for Researchers**
    - RESTful API with rate limiting for academic and civil society access
    - Anonymized datasets for research purposes
    - Documentation and SDKs for common programming languages

### Scalability Roadmap

1. **Geographic Expansion**
   - Municipal and local government procurement (panchayat level)
   - Public sector undertakings and autonomous bodies
   - State-specific customization for regional procurement rules

2. **Data Integration**
   - Asset declaration systems for conflict of interest detection
   - Corporate registry integration for beneficial ownership analysis
   - News and media monitoring for correlation with flagged cases

3. **Real-time Monitoring**
   - Webhook subscriptions for instant alerts on high-risk tenders
   - Mobile push notifications for watched departments
   - Automated daily/weekly digest emails with risk summaries

4. **Offline Capabilities**
   - Mobile app with offline-first architecture for low-connectivity regions
   - Sync mechanism for data updates when connection available
   - Lightweight data formats optimized for bandwidth constraints

5. **Official Integration**
   - Partnership with government for official transparency portal integration
   - Feedback loop for departments to improve procurement practices
   - Training programs for procurement officials on transparency best practices

### Research Directions

1. **Causal Inference**: Move beyond correlation to establish causal relationships between risk patterns and actual corruption
2. **Fairness Analysis**: Ensure algorithms don't discriminate against small vendors or specific regions
3. **Adversarial Robustness**: Detect and prevent gaming of risk detection systems
4. **Explainable AI**: Advanced interpretability techniques (SHAP, LIME) for complex ML models
