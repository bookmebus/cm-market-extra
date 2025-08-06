# XFactor Cambodia Integration Specifications

## Overview
Special features and integrations for XFactor Cambodia within the Hang Meas ticketing app.

## XFactor-Specific Features

### 1. Live Voting System

#### Voting Mechanics
- Real-time voting during live shows
- Multiple voting methods:
  - In-app voting (free, limited votes)
  - SMS voting (premium)
  - Vote packages (bulk votes purchase)
- Vote validation and fraud prevention
- Geographic restrictions (Cambodia only)

#### Technical Implementation
```javascript
// Voting API structure
POST /api/v1/xfactor/vote
{
  "contestant_id": "uuid",
  "episode_id": "uuid",
  "vote_type": "free|sms|package",
  "vote_count": 1,
  "device_id": "string",
  "location": {
    "lat": 11.5564,
    "lng": 104.9282
  }
}
```

### 2. Contestant Management

#### Features
- Contestant profiles with photos/videos
- Performance history
- Judge comments and scores
- Fan comments and reactions
- Social media integration

#### Data Structure
```javascript
{
  "contestant": {
    "id": "uuid",
    "name": "string",
    "age": 20,
    "hometown": "string",
    "bio_km": "string",
    "bio_en": "string",
    "photo_url": "string",
    "audition_video": "string",
    "performances": [...],
    "social_media": {
      "facebook": "url",
      "instagram": "handle",
      "tiktok": "handle"
    }
  }
}
```

### 3. Show Schedule & Tickets

#### Ticket Types
- **Single Show Tickets**
  - Studio audience tickets
  - VIP meet & greet packages
  
- **Season Pass**
  - Access to all live shows
  - Priority seating
  - Exclusive content access
  - Voting benefits

#### Special Features
- Virtual tickets for live streaming
- Bundle deals with voting packages
- Early bird discounts
- Group bookings for fan clubs

### 4. Interactive Features

#### During Live Shows
- Real-time voting results
- Live polls and trivia
- Chat with other viewers
- Emoji reactions
- Judge prediction games

#### Between Shows
- Behind-the-scenes content
- Contestant video diaries
- Rehearsal footage
- Exclusive interviews
- Weekly challenges

### 5. Gamification & Rewards

#### Fan Engagement System
- Points for various activities:
  - Watching live shows
  - Voting regularly
  - Sharing content
  - Accurate predictions
  
#### Rewards
- Exclusive content unlocks
- Meet & greet opportunities
- Signed merchandise
- Priority ticket access
- Vote multipliers

## Technical Architecture

### Real-time Components
- WebSocket for live voting
- Server-Sent Events for result updates
- Redis for vote aggregation
- Elasticsearch for real-time analytics

### Scalability Considerations
- Microservices architecture for voting system
- Separate database for voting data
- CDN for video content
- Auto-scaling for peak voting periods

### Anti-Fraud Measures
- Device fingerprinting
- IP rate limiting
- Geographic verification
- CAPTCHA for suspicious activity
- Machine learning fraud detection

## Integration Points

### 1. Broadcasting System
- API integration with TV broadcast system
- Synchronized voting windows
- Real-time result display on TV

### 2. SMS Gateway
- Premium SMS voting
- Vote confirmation messages
- Balance checking
- Vote package purchases

### 3. Social Media
- Share voting results
- Contestant support campaigns
- Social login for voting
- Content syndication

## Monetization Opportunities

### Vote Packages
```
- Starter Pack: 10 votes - $0.99
- Fan Pack: 50 votes - $3.99
- Super Fan Pack: 200 votes - $9.99
- Mega Pack: 1000 votes - $39.99
```

### Premium Features
- Ad-free experience
- HD streaming quality
- Exclusive camera angles
- Judge's extended comments
- Download performances

### Sponsored Content
- Brand integration in voting
- Sponsored challenges
- Product placement rewards
- Corporate voting packages

## Analytics & Reporting

### Real-time Dashboards
- Live voting statistics
- Geographic distribution
- Demographic analysis
- Revenue tracking
- Engagement metrics

### Post-Show Analysis
- Voting patterns
- Contestant popularity trends
- Revenue per episode
- User retention metrics
- Social media impact

## Content Management

### Admin Panel Features
- Contestant profile management
- Performance video uploads
- Voting window controls
- Result moderation
- Content scheduling
- Push notification management

### Moderation Tools
- Comment filtering
- User reporting system
- Automated content moderation
- Manual review queue
- Ban/suspension system

## Marketing Integration

### Campaign Features
- Contestant promotional tools
- Shareable voting cards
- Referral rewards
- Social media templates
- Email marketing integration

### Partnership Opportunities
- Telecom operator deals
- Bank promotion integration
- Brand sponsorship slots
- Media partner benefits