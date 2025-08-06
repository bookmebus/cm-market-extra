# Technical Requirements for Hang Meas Ticketing App

## Mobile App Requirements

### Platform Support
- iOS 13.0+ (iPhone & iPad)
- Android 6.0+ (API level 23)
- Responsive design for various screen sizes

### Performance Requirements
- App launch time < 3 seconds
- Page load time < 2 seconds
- Offline capability for viewing purchased tickets
- Support for 10,000+ concurrent users during ticket launches

### User Interface
- Khmer and English language support
- Dark/Light mode
- Accessibility compliance (WCAG 2.1)
- Smooth animations and transitions

## Backend API Requirements

### API Design
```
/api/v1/
├── auth/
│   ├── register
│   ├── login
│   ├── refresh
│   └── logout
├── users/
│   ├── profile
│   ├── tickets
│   └── preferences
├── events/
│   ├── list
│   ├── details/{id}
│   ├── seats/{id}
│   └── categories
├── tickets/
│   ├── purchase
│   ├── verify
│   └── cancel
├── payments/
│   ├── methods
│   ├── process
│   └── status
└── xfactor/
    ├── contestants
    ├── vote
    └── results
```

### Database Schema

#### Users Table
```sql
- id (UUID)
- email
- phone_number
- full_name
- password_hash
- language_preference
- created_at
- updated_at
```

#### Events Table
```sql
- id (UUID)
- title_km
- title_en
- description_km
- description_en
- venue_id
- event_date
- sale_start_date
- sale_end_date
- status
- category
- featured
- created_at
```

#### Tickets Table
```sql
- id (UUID)
- event_id
- user_id
- ticket_type
- seat_number
- price
- status
- qr_code
- purchase_date
- created_at
```

#### XFactor Voting Table
```sql
- id (UUID)
- user_id
- contestant_id
- episode_id
- vote_time
- ip_address
```

## Integration Specifications

### Payment Gateway Requirements
- Support for multiple currencies (USD, KHR)
- Transaction timeout: 5 minutes
- Webhook for payment status updates
- PCI DSS compliance
- Transaction logs retention: 7 years

### SMS Gateway
- Delivery rate > 99%
- Support for Khmer Unicode
- Bulk SMS capability
- Delivery reports
- Rate limit: 100 SMS/second

### QR Code Generation
- Format: QR Code Model 2
- Error correction level: M (15%)
- Size: 300x300 pixels minimum
- Embedded data: ticket ID, event ID, validation hash

## Security Requirements

### Authentication
- JWT with 15-minute access token expiry
- Refresh tokens with 30-day expiry
- Multi-factor authentication option
- Account lockout after 5 failed attempts

### Data Protection
- TLS 1.3 for all communications
- AES-256 encryption for sensitive data
- Bcrypt for password hashing
- API rate limiting: 100 requests/minute per user

### Compliance
- GDPR compliance for EU users
- Local data privacy laws
- Payment Card Industry (PCI) compliance
- Regular security audits

## Monitoring & Analytics

### Application Monitoring
- Crash reporting (Firebase Crashlytics)
- Performance monitoring
- User session recording (with consent)
- API response time tracking

### Business Analytics
- Google Analytics or similar
- Custom event tracking
- Revenue analytics
- User behavior flow
- A/B testing capability

## Development Environment

### Version Control
- Git with GitFlow workflow
- Feature branches
- Code review requirement
- Automated CI/CD pipeline

### Testing Requirements
- Unit test coverage > 80%
- Integration tests for all APIs
- End-to-end testing for critical flows
- Performance testing for high-load scenarios
- Security penetration testing

### Documentation
- API documentation (OpenAPI/Swagger)
- Code documentation
- Deployment guides
- User manuals in Khmer and English