# SmartMandi AI - Requirements Document

## 1. Product Overview

SmartMandi AI is a cloud-native market intelligence and decision-support system designed to empower farmers in India with data-driven insights for crop selling decisions. Built on AWS infrastructure, the platform forecasts mandi (APMC market) crop prices using machine learning and delivers actionable recommendations through multiple channels including SMS, IVR voice calls, and a web dashboard.

The system is specifically designed for low-connectivity rural environments and ensures accessibility for farmers without smartphones or internet access.

## 2. Problem Statement

Indian farmers face significant challenges in making informed crop selling decisions:

- **Information Asymmetry**: Lack of real-time price intelligence across different mandis leads to suboptimal selling decisions
- **Market Volatility**: Unpredictable price fluctuations result in financial losses
- **Limited Access**: Rural farmers often lack smartphones or reliable internet connectivity
- **Language Barriers**: Digital solutions often don't cater to regional languages
- **Time Sensitivity**: Delayed decisions can result in crop spoilage and reduced profits

Farmers need a simple, accessible system that provides timely price forecasts and actionable recommendations to maximize their returns.

## 3. Goals & Objectives

### Primary Goals
- Provide accurate 7-day mandi price forecasts for major crops
- Deliver actionable sell/hold recommendations based on predicted price trends
- Suggest optimal mandis for selling based on price and proximity
- Ensure accessibility for farmers without smartphones or internet

### Business Objectives
- Increase farmer income by 10-15% through better selling decisions
- Achieve 80%+ forecast accuracy for major crops
- Support 10,000+ farmers in MVP phase
- Maintain system availability of 99.5%+
- Keep operational costs under ₹5 per farmer per month

## 4. Target Users / Personas

### Persona 1: Traditional Farmer (Primary)
- **Name**: Ramesh Kumar
- **Age**: 45-60
- **Location**: Rural Maharashtra
- **Tech**: Basic feature phone, no internet
- **Language**: Hindi/Marathi
- **Needs**: Simple voice/SMS updates, regional language support
- **Crops**: Cotton, Soybean, Wheat

### Persona 2: Progressive Farmer (Secondary)
- **Name**: Priya Sharma
- **Age**: 28-40
- **Location**: Semi-urban Punjab
- **Tech**: Smartphone with intermittent internet
- **Language**: Hindi/Punjabi/English
- **Needs**: Web dashboard, detailed analytics, historical trends
- **Crops**: Rice, Wheat, Vegetables

### Persona 3: Agricultural Extension Officer (Tertiary)
- **Name**: Dr. Suresh Patel
- **Role**: Government agricultural advisor
- **Tech**: Laptop/smartphone with good connectivity
- **Needs**: Bulk farmer management, regional insights, reporting tools

## 5. Scope

### In Scope
- ML-based price forecasting for top 20 crops across major mandis
- SMS-based price alerts and recommendations
- IVR voice call system for recommendations in regional languages
- Web dashboard for detailed analytics and historical data
- User registration and preference management
- Integration with Agmarknet public data
- Weather data integration for forecast accuracy
- Multi-language support (Hindi, English, + 3 regional languages)
- Basic authentication and user management

### Out of Scope
- Ecommerce or marketplace functionality
- Direct farmer-to-buyer transactions
- Payment processing or financial transactions
- Crop cultivation advice or farming techniques
- Pest/disease management
- Soil testing or analysis
- Mobile native applications (MVP phase)
- Real-time commodity trading
- Loan or credit facilities
- Insurance products

## 6. Functional Requirements

| ID | Requirement | Priority | Category |
|----|-------------|----------|----------|
| FR-001 | System shall ingest daily mandi price data from Agmarknet | High | Data Ingestion |
| FR-002 | System shall ingest weather data from IMD/weather APIs | High | Data Ingestion |
| FR-003 | System shall store historical price data for minimum 3 years | High | Data Storage |
| FR-004 | System shall train ML models to forecast prices for 7-day horizon | High | ML/AI |
| FR-005 | System shall generate sell/hold recommendations based on price trends | High | ML/AI |
| FR-006 | System shall identify top 3 optimal mandis based on price and distance | High | ML/AI |
| FR-007 | System shall send daily SMS alerts to registered farmers | High | Notifications |
| FR-008 | System shall support IVR calls in 5+ regional languages | High | Notifications |
| FR-009 | System shall provide web dashboard with price charts and trends | Medium | UI/UX |
| FR-010 | System shall allow farmer registration via SMS, IVR, or web | High | User Management |
| FR-011 | System shall support user preferences (crops, mandis, language) | Medium | User Management |
| FR-012 | System shall provide historical price comparison (YoY, MoM) | Medium | Analytics |
| FR-013 | System shall generate weekly summary reports | Low | Reporting |
| FR-014 | System shall support admin panel for system monitoring | Medium | Administration |
| FR-015 | System shall log all predictions for accuracy tracking | High | Monitoring |
| FR-016 | System shall retrain models weekly with new data | High | ML/AI |
| FR-017 | System shall provide API endpoints for third-party integrations | Low | Integration |
| FR-018 | System shall support bulk farmer onboarding via CSV upload | Medium | User Management |
| FR-019 | System shall send alerts only during farmer-preferred time windows | Medium | Notifications |
| FR-020 | System shall provide forecast confidence scores | Medium | ML/AI |

## 7. Non-Functional Requirements

### Performance
- **NFR-001**: API response time < 500ms for 95th percentile
- **NFR-002**: ML inference latency < 2 seconds per prediction
- **NFR-003**: SMS delivery within 30 seconds of trigger
- **NFR-004**: IVR call connection time < 10 seconds
- **NFR-005**: Web dashboard load time < 3 seconds

### Scalability
- **NFR-006**: Support 100,000 farmers without architecture changes
- **NFR-007**: Handle 10,000 concurrent SMS requests
- **NFR-008**: Process 500+ mandi price updates daily
- **NFR-009**: Auto-scale Lambda functions based on demand

### Availability & Reliability
- **NFR-010**: System uptime of 99.5% (excluding planned maintenance)
- **NFR-011**: Zero data loss for price ingestion
- **NFR-012**: Automated failover for critical services
- **NFR-013**: Daily automated backups with 30-day retention

### Security
- **NFR-014**: All data encrypted at rest (S3, databases)
- **NFR-015**: All data encrypted in transit (TLS 1.2+)
- **NFR-016**: Phone number masking in logs and analytics
- **NFR-017**: Role-based access control (RBAC) for admin functions
- **NFR-018**: API authentication using AWS Cognito
- **NFR-019**: Compliance with Indian data protection regulations
- **NFR-020**: Regular security audits and vulnerability scanning

### Cost Optimization
- **NFR-021**: Operational cost < ₹5 per farmer per month
- **NFR-022**: Use serverless architecture to minimize idle costs
- **NFR-023**: Implement S3 lifecycle policies for data archival
- **NFR-024**: Optimize ML training costs using spot instances

### Usability
- **NFR-025**: SMS messages limited to 160 characters
- **NFR-026**: IVR menu depth limited to 3 levels
- **NFR-027**: Web dashboard accessible on 2G connections
- **NFR-028**: Support for screen readers (WCAG 2.1 Level A)

### Maintainability
- **NFR-029**: Infrastructure as Code using AWS CDK/CloudFormation
- **NFR-030**: Comprehensive logging using CloudWatch
- **NFR-031**: Automated CI/CD pipeline for deployments
- **NFR-032**: API versioning for backward compatibility

## 8. Data Requirements

### Data Sources

#### Primary Data
- **Agmarknet Mandi Prices**
  - Source: https://agmarknet.gov.in
  - Frequency: Daily
  - Coverage: 3000+ mandis, 300+ commodities
  - Format: CSV/API
  - Historical: 3+ years

- **Weather Data**
  - Source: IMD API or OpenWeatherMap
  - Frequency: Daily
  - Parameters: Temperature, rainfall, humidity
  - Coverage: District-level

#### Optional Data
- Crop production statistics (Ministry of Agriculture)
- Festival/holiday calendar (affects demand)
- Transportation cost indices

### Data Storage

| Data Type | Storage | Retention | Size Estimate |
|-----------|---------|-----------|---------------|
| Raw mandi prices | S3 (Standard) | 3 years | 10 GB/year |
| Processed features | S3 (Standard) | 1 year | 5 GB/year |
| ML models | S3 (Standard) | All versions | 500 MB |
| User profiles | DynamoDB | Active users | 100 MB |
| Predictions | DynamoDB | 90 days | 1 GB |
| Logs | CloudWatch | 30 days | 5 GB/month |
| Archived data | S3 (Glacier) | 7 years | 50 GB |

### Data Quality Requirements
- Price data completeness > 95% for major mandis
- Weather data accuracy validated against IMD standards
- Duplicate detection and removal in ingestion pipeline
- Outlier detection for anomalous price spikes
- Data validation rules for all inputs

## 9. User Stories

### Farmer Stories

**US-001**: As a farmer, I want to receive daily SMS alerts about predicted prices for my crops, so I can plan when to sell.

**US-002**: As a farmer without a smartphone, I want to call an IVR number to hear price forecasts in my language, so I can make informed decisions.

**US-003**: As a farmer, I want to know whether to sell today or wait, so I can maximize my profits.

**US-004**: As a farmer, I want to see which nearby mandis offer the best prices, so I can choose where to sell.

**US-005**: As a farmer, I want to register using my phone number via SMS, so I can start receiving alerts without internet.

**US-006**: As a farmer, I want to select which crops I grow, so I only receive relevant price information.

**US-007**: As a progressive farmer, I want to view historical price trends on a dashboard, so I can understand seasonal patterns.

**US-008**: As a farmer, I want to receive alerts only during morning hours, so I'm not disturbed at night.

### Extension Officer Stories

**US-009**: As an extension officer, I want to onboard multiple farmers at once, so I can help my community access the service.

**US-010**: As an extension officer, I want to view regional price trends, so I can provide better guidance to farmers.

### Admin Stories

**US-011**: As a system admin, I want to monitor forecast accuracy, so I can improve the ML models.

**US-012**: As a system admin, I want to view system health metrics, so I can ensure reliable service delivery.

**US-013**: As a system admin, I want to manage user subscriptions, so I can handle support requests.

## 10. Success Metrics / KPIs

### Business Metrics
- **Farmer Adoption**: 10,000+ registered farmers in 6 months
- **Engagement Rate**: 60%+ farmers actively using recommendations
- **Income Impact**: 10-15% increase in farmer selling prices
- **User Satisfaction**: NPS score > 50

### Technical Metrics
- **Forecast Accuracy**: MAPE < 15% for 7-day forecasts
- **System Uptime**: 99.5%+ availability
- **SMS Delivery Rate**: 95%+ successful deliveries
- **IVR Completion Rate**: 70%+ calls completed
- **API Latency**: p95 < 500ms

### Operational Metrics
- **Cost per Farmer**: < ₹5/month
- **Data Freshness**: Price data updated within 6 hours of mandi close
- **Model Retraining**: Weekly automated retraining
- **Support Tickets**: < 5% of active users per month

### Growth Metrics
- **Month-over-Month Growth**: 20%+ new farmer registrations
- **Retention Rate**: 80%+ farmers active after 3 months
- **Geographic Coverage**: 10+ districts in MVP phase

## 11. Constraints & Assumptions

### Technical Constraints
- Must use only AWS services for infrastructure
- Cannot use proprietary or paid data sources
- SMS costs limited to ₹0.20 per message
- IVR costs limited to ₹0.50 per minute
- Must work on 2G network speeds

### Business Constraints
- MVP must be completed within hackathon timeline
- Limited budget for initial deployment
- No dedicated mobile app development resources
- Dependency on public data availability and quality

### Regulatory Constraints
- Compliance with Indian telecom regulations (TRAI)
- Data privacy compliance (IT Act 2000)
- No financial advice or guarantees on prices
- Disclaimer required for all recommendations

### Assumptions
- Agmarknet data will remain publicly accessible
- Farmers have access to basic feature phones
- Phone numbers provided are valid and active
- Farmers understand basic market concepts (mandi, price trends)
- Regional language translations are accurate
- Weather data correlates with price movements
- Historical patterns are indicative of future trends
- Farmers can travel to recommended mandis within reasonable distance
- SMS and voice calls are preferred communication channels
- Internet connectivity will improve gradually in rural areas

## 12. Risks & Mitigation

| Risk | Impact | Probability | Mitigation Strategy |
|------|--------|-------------|---------------------|
| Agmarknet API downtime | High | Medium | Cache data, implement retry logic, use backup sources |
| Poor ML model accuracy | High | Medium | Ensemble models, regular retraining, human-in-loop validation |
| High SMS/IVR costs | Medium | High | Optimize message frequency, use tiered alerts, negotiate bulk rates |
| Low farmer adoption | High | Medium | Partner with NGOs, government schemes, local influencers |
| Data quality issues | Medium | High | Implement robust validation, outlier detection, manual review |
| Language translation errors | Medium | Low | Use professional translators, farmer feedback loop |
| Scalability bottlenecks | Medium | Low | Load testing, auto-scaling, serverless architecture |
| Security breaches | High | Low | AWS security best practices, regular audits, encryption |
| Regulatory changes | Medium | Low | Legal consultation, flexible architecture, compliance monitoring |
| Weather API costs | Low | Medium | Use free tier, cache data, optimize API calls |
| Farmer phone number changes | Low | High | Periodic verification, easy update mechanism |
| Seasonal data gaps | Medium | Medium | Multi-year training data, synthetic data augmentation |

## 13. MVP Scope for Hackathon

### Core Features (Must Have)
1. **Data Pipeline**
   - Automated Agmarknet data ingestion (daily)
   - Weather data integration
   - S3-based data lake

2. **ML Forecasting**
   - Price prediction for top 5 crops (Wheat, Rice, Cotton, Soybean, Onion)
   - 7-day forecast horizon
   - Basic LSTM/Prophet model
   - Coverage: 50 major mandis

3. **Recommendation Engine**
   - Sell/hold decision logic
   - Top 3 mandi suggestions

4. **Notification System**
   - SMS alerts using SNS
   - Basic IVR using Amazon Connect (Hindi + English)
   - Daily scheduled notifications

5. **Web Dashboard**
   - Farmer registration
   - Price charts and forecasts
   - Historical trends
   - Hosted on Amplify

6. **User Management**
   - Basic authentication (Cognito)
   - User preferences (crops, mandis)
   - Phone number verification

### Deferred to Post-MVP
- Advanced ML models (ensemble, deep learning)
- 5+ regional languages
- Mobile-optimized PWA
- Admin analytics dashboard
- Bulk farmer onboarding
- API for third-party integrations
- Advanced personalization
- Feedback collection system

### MVP Success Criteria
- System successfully forecasts prices for 5 crops
- 100 test farmers registered and receiving alerts
- 80%+ SMS delivery rate
- Working IVR demo in 2 languages
- Web dashboard accessible and functional
- End-to-end demo ready for hackathon presentation

## 14. Future Enhancements

### Phase 2 (Post-Hackathon)
- Expand to 20+ crops and 200+ mandis
- Add 5 more regional languages
- Implement advanced ML models (XGBoost, ensemble)
- Build admin analytics dashboard
- Add farmer feedback mechanism
- Integrate transportation cost data
- Implement A/B testing for recommendations

### Phase 3 (6-12 Months)
- Progressive Web App (PWA) for offline access
- WhatsApp Business API integration
- Personalized recommendations based on farm size and location
- Community features (farmer forums, success stories)
- Integration with government schemes (PM-KISAN, etc.)
- Crop advisory based on predicted prices
- Market demand forecasting

### Phase 4 (12+ Months)
- AI-powered chatbot for farmer queries
- Blockchain for price transparency
- Integration with digital payment systems
- Satellite imagery for crop health monitoring
- Supply chain optimization recommendations
- Predictive analytics for crop selection
- Partnership with agri-input companies
- Export market price integration

### Long-term Vision
- Expand to other agricultural markets (livestock, dairy)
- Pan-India coverage (all states and major mandis)
- Multi-modal AI (voice, image, text)
- Farmer credit scoring based on selling patterns
- Integration with FPOs and cooperatives
- Real-time market sentiment analysis
- Climate change impact modeling

---

**Document Version**: 1.0  
**Last Updated**: February 5, 2026  
**Owner**: SmartMandi AI Team  
**Status**: Draft for Hackathon Submission
