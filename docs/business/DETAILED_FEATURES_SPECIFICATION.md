# Detailed Features Specification - Hang Meas Ticketing App

> **Multi-Show Platform**: This application supports multiple talent shows through a unified framework. See [TALENT_SHOWS_FRAMEWORK.md](../TALENT_SHOWS_FRAMEWORK.md) for the complete multi-show architecture.

## Table of Contents
1. [User Management System](#1-user-management-system)
2. [Event Discovery & Browsing](#2-event-discovery--browsing)
3. [Ticket Purchase Flow](#3-ticket-purchase-flow)
4. [Multi-Show Live Voting](#4-multi-show-live-voting)
5. [Payment Processing](#5-payment-processing)
6. [Digital Ticket Management](#6-digital-ticket-management)
7. [Push Notifications](#7-push-notifications)
8. [Admin Dashboard](#8-admin-dashboard)

---

## 1. User Management System

### 1.1 Registration Process

#### Phone Number Registration (Primary Method)
**Flow Example:**
1. User enters phone number: `+855 12 345 678`
2. System sends OTP via SMS: "Your Hang Meas verification code is: 5847"
3. User enters OTP
4. User completes profile:
   - Full name (Khmer & English)
   - Date of birth
   - Gender
   - Location (Province/City)
   - Password creation

**Screen Examples:**
```
┌─────────────────────┐
│   Create Account    │
│                     │
│ 📱 Phone Number     │
│ +855 ____________   │
│                     │
│ [Send OTP]          │
│                     │
│ Already have        │
│ account? Sign in    │
└─────────────────────┘
```

#### Social Media Registration
**Supported Platforms:**
- Facebook (most popular in Cambodia)
- Google Account
- Apple ID (iOS only)

**Data Retrieved:**
- Name
- Email (if available)
- Profile picture
- Basic demographics

### 1.2 User Profile Features

#### Profile Information
```javascript
{
  "user_profile": {
    "basic_info": {
      "name_km": "សុខ សុភា",
      "name_en": "Sok Sophia",
      "phone": "+855123456789",
      "email": "sophia@example.com",
      "dob": "1995-05-15",
      "gender": "female",
      "profile_picture": "url_to_image"
    },
    "preferences": {
      "language": "km", // km or en
      "favorite_artists": ["Preap Sovath", "Sokun Nisa"],
      "favorite_venues": ["Koh Pich Theatre", "Olympic Stadium"],
      "notification_settings": {
        "new_events": true,
        "price_drops": true,
        "artist_updates": true,
        "talent_show_voting": true
      }
    },
    "loyalty_points": 1250,
    "member_since": "2024-01-15"
  }
}
```

#### Purchase History
**Display Format:**
```
┌─────────────────────────┐
│   My Tickets            │
├─────────────────────────┤
│ Upcoming Events         │
│                         │
│ 🎤 Preap Sovath Concert │
│ 📅 Dec 25, 2024 7:00PM  │
│ 🎫 2 VIP tickets        │
│ 📍 Koh Pich Theatre     │
│ [View Tickets]          │
│                         │
│ 🎭 Voice Cambodia Final │
│ 📅 Jan 15, 2025 8:00PM  │
│ 🎫 4 Standard tickets   │
│ 📍 Olympic Stadium      │
│ [View Tickets]          │
├─────────────────────────┤
│ Past Events             │
│ View all (23) >         │
└─────────────────────────┘
```

---

## 2. Event Discovery & Browsing

### 2.1 Home Screen Layout

**Featured Section:**
```
┌─────────────────────────┐
│ 🔍 Search events...     │
├─────────────────────────┤
│ Featured This Week      │
│ ┌───────┐ ┌───────┐     │
│ │       │ │       │     │
│ │XFactor│ │Concert│  ►  │
│ │ Final │ │Sovath │     │
│ └───────┘ └───────┘     │
├─────────────────────────┤
│ Categories              │
│ 🎤 Concerts  🎭 Shows    │
│ 🎭 Talent Shows 🎉 Events │
├─────────────────────────┤
│ Upcoming Events         │
│ ▼ December 2024         │
└─────────────────────────┘
```

### 2.2 Event Categories

#### Concert Category
**Sub-categories:**
- Traditional Khmer Music
- Modern Khmer Pop
- International Artists
- Music Festivals
- Charity Concerts

**Example Event Card:**
```
┌─────────────────────────┐
│ [Event Banner Image]    │
│                         │
│ Khmer New Year Concert  │
│ Multiple Artists        │
│                         │
│ 📅 April 14-16, 2025    │
│ 📍 Angkor Wat           │
│ 💵 $10 - $100           │
│                         │
│ ⭐ 4.8 (234 reviews)    │
│ 🔥 80% sold             │
└─────────────────────────┘
```

#### Talent Shows Category
**Supported Shows:**
- **XFactor Cambodia**: Singing competition with judge categories
  - Auditions, Boot Camp, Six Chair Challenge, Live Shows, Final
- **Cambodia's Got Talent**: Variety acts with Golden Buzzer
  - Open Auditions, Judge Auditions, Semi-Finals, Finals
- **The Voice Cambodia**: Blind auditions with coach teams
  - Blind Auditions, Battles, Knockouts, Live Shows, Final

> **Show-Specific Features**: Each show has unique voting rules and features detailed in [TALENT_SHOWS_FRAMEWORK.md](../TALENT_SHOWS_FRAMEWORK.md)

### 2.3 Search & Filter System

**Filter Options:**
```javascript
{
  "filters": {
    "date_range": {
      "from": "2024-12-01",
      "to": "2024-12-31"
    },
    "price_range": {
      "min": 0,
      "max": 100,
      "currency": "USD"
    },
    "location": {
      "province": "Phnom Penh",
      "venues": ["Koh Pich Theatre", "Olympic Stadium"]
    },
    "event_type": ["concert", "xfactor", "cgt", "voice"],
    "availability": "available_only",
    "language": "km", // Event language
    "features": [
      "parking_available",
      "food_allowed",
      "vip_section",
      "live_streaming"
    ]
  }
}
```

**Search Examples:**
- "សុខ សុភា" (Artist name in Khmer)
- "December concerts"
- "XFactor voting night"
- "Cambodia's Got Talent golden buzzer"
- "Voice blind auditions"
- "Events near me"

---

## 3. Ticket Purchase Flow

### 3.1 Event Details Page

**Information Display:**
```
┌─────────────────────────┐
│ [Hero Image/Video]      │
│ ▶️                      │
├─────────────────────────┤
│ Preap Sovath Live       │
│ ⭐⭐⭐⭐⭐ (456)          │
│                         │
│ 📅 Dec 25, 2024         │
│ ⏰ 7:00 PM - 10:00 PM   │
│ 📍 Koh Pich Theatre     │
│                         │
│ About This Event        │
│ Cambodia's legendary    │
│ singer returns with...  │
│ [Read more]             │
│                         │
│ Ticket Types            │
│ ┌─────────────────────┐ │
│ │ VIP Front Row       │ │
│ │ $100 per ticket     │ │
│ │ ✓ Meet & Greet      │ │
│ │ ✓ Premium Seating   │ │
│ │ [Select]            │ │
│ ├─────────────────────┤ │
│ │ Standard            │ │
│ │ $30 per ticket      │ │
│ │ ✓ General Seating   │ │
│ │ [Select]            │ │
│ └─────────────────────┘ │
└─────────────────────────┘
```

### 3.2 Seat Selection (Seated Venues)

**Interactive Seat Map:**
```
         STAGE
    ┌─────────────┐
    
A   ⬜⬜⬛⬛⬜⬜⬜⬜   VIP
B   ⬜⬛⬛⬛⬛⬜⬜⬜   $100
C   ⬜⬜⬜⬛⬜⬜⬜⬜
    
D   ⬜⬜⬜⬜⬜⬜⬜⬜   Standard
E   ⬛⬛⬜⬜⬜⬜⬛⬛   $30
F   ⬜⬜⬜⬜⬜⬜⬜⬜

Legend:
⬜ Available
⬛ Sold
🟦 Selected
```

**Seat Information:**
- Real-time availability
- Price per seat
- View from seat (photo)
- Accessibility options

### 3.3 Checkout Process

**Step 1: Review Order**
```
┌─────────────────────────┐
│ Order Summary           │
├─────────────────────────┤
│ Preap Sovath Live       │
│ Dec 25, 2024 7:00 PM    │
│                         │
│ 2x VIP Front Row        │
│ Seats: A4, A5           │
│ $100 x 2 = $200         │
│                         │
│ Service Fee (5%)  $10   │
│ ─────────────────────   │
│ Total:           $210   │
│                         │
│ [Apply Promo Code]      │
│                         │
│ [Continue to Payment]   │
└─────────────────────────┘
```

**Step 2: Payment Selection**
```
┌─────────────────────────┐
│ Select Payment Method   │
├─────────────────────────┤
│ 🏦 Local Banks          │
│ ┌─────────────────────┐ │
│ │ ABA Bank            │ │
│ │ Quick pay with QR   │ │
│ └─────────────────────┘ │
│ ┌─────────────────────┐ │
│ │ ACLEDA Bank         │ │
│ │ Internet banking    │ │
│ └─────────────────────┘ │
├─────────────────────────┤
│ 📱 Mobile Payments      │
│ ┌─────────────────────┐ │
│ │ Wing                │ │
│ │ Pay with phone #    │ │
│ └─────────────────────┘ │
│ ┌─────────────────────┐ │
│ │ TrueMoney           │ │
│ │ Wallet payment      │ │
│ └─────────────────────┘ │
├─────────────────────────┤
│ 💳 Cards                │
│ ┌─────────────────────┐ │
│ │ Visa/Mastercard     │ │
│ │ International cards │ │
│ └─────────────────────┘ │
└─────────────────────────┘
```

---

## 4. Multi-Show Live Voting

> **Framework Overview**: The voting system supports multiple talent shows with different rules and mechanics. See [TALENT_SHOWS_FRAMEWORK.md](../TALENT_SHOWS_FRAMEWORK.md) for detailed implementation.

### 4.1 Dynamic Voting Interface

**The interface adapts based on the active show type:**

#### XFactor Voting Screen:
```
┌─────────────────────────┐
│ 🔴 LIVE - XFactor       │
│ Semi-Final Round 2      │
│ Voting closes in: 4:32  │
├─────────────────────────┤
│ Cast Your Vote          │
│                         │
│ ┌─────────────────────┐ │
│ │ 👤 Contestant 1     │ │
│ │ Dara - Boys Category │ │
│ │ Song: "អូនស្រលាញ់"    │ │
│ │ [VOTE] ❤️ 23.5K     │ │
│ └─────────────────────┘ │
│                         │
│ Your Votes: 3/5 FREE    │
│ [Buy More Votes]        │
└─────────────────────────┘
```

#### Cambodia's Got Talent Voting:
```
┌─────────────────────────┐
│ 🔴 LIVE - CGT Finals    │
│ Voting closes in: 8:15  │
├─────────────────────────┤
│ Vote for Your Favorite  │
│                         │
│ ┌─────────────────────┐ │
│ │ 🎪 Magic Act        │ │
│ │ Seila - Battambang  │ │
│ │ 🏆 Golden Buzzer    │ │
│ │ [VOTE] ⭐ 18.3K     │ │
│ └─────────────────────┘ │
│                         │
│ Your Votes: 7/10 FREE   │
│ [Buy More Votes]        │
└─────────────────────────┘
```

#### The Voice Voting:
```
┌─────────────────────────┐
│ 🔴 LIVE - Voice Final   │
│ Voting closes in: 5:45  │
├─────────────────────────┤
│ Vote by Coach Teams     │
│                         │
│ ┌─────────────────────┐ │
│ │ 🎤 Team Preap       │ │
│ │ Kimly - Kandal      │ │
│ │ Song: "ស្នេហាអើយ"     │ │
│ │ [VOTE] 💙 15.7K     │ │
│ └─────────────────────┘ │
│                         │
│ Instant Save Available  │
│ [Buy More Votes]        │
└─────────────────────────┘
```

### 4.2 Multi-Show Vote Packages

**The vote packages adapt to the active show:**

#### Universal Vote Store:
```
┌─────────────────────────┐
│ Talent Show Voting      │
│ Currently: XFactor Live │
├─────────────────────────┤
│ 🎯 Basic Package        │
│ 20 votes                │
│ $1.99 - All shows       │
├─────────────────────────┤
│ 🌟 Power Vote Bundle    │
│ 50 votes + bonus 10     │
│ $4.99 SAVE 20%          │
│ [Most Popular]          │
├─────────────────────────┤
│ 💎 Super Fan Package    │
│ 200 votes + bonuses     │
│ $14.99 SAVE 30%         │
│ + Show-specific perks:  │
│   • XFactor: Categories │
│   • CGT: Golden predict │
│   • Voice: Team loyalty │
└─────────────────────────┘
```

**Show-Specific Benefits:**
- **XFactor**: Vote for entire categories, mentor content
- **CGT**: Predict Golden Buzzer usage, act-type voting
- **Voice**: Team packages, coach cam access

### 4.3 Voting Mechanics

**Vote Types:**

1. **Free Votes**
   - 5 free votes per episode
   - Must watch 80% of show
   - One vote per contestant max
   - Reset each episode

2. **Paid Votes**
   - Unlimited purchases
   - Can vote multiple times for same contestant
   - Bulk voting option
   - Carry over to next episode if unused

3. **SMS Votes**
   - Send contestant number to 1255
   - $0.50 per vote
   - Confirmation SMS sent
   - Added to phone bill

**Universal Voting API:**
```javascript
// Multi-show voting endpoint
POST /api/v1/shows/:showType/vote
{
  "show_type": "xfactor", // or "cgt", "voice"
  "contestant_id": "cont_123",
  "episode_id": "ep_sf2",
  "votes": [
    {
      "type": "free",
      "count": 1
    }
  ],
  "show_specific_data": {
    "category": "boys", // XFactor specific
    "coach_team": "preap" // Voice specific
  },
  "timestamp": "2024-12-20T20:15:30Z",
  "device_fingerprint": "abc123..."
}

// Universal Response Format
{
  "success": true,
  "vote_id": "vote_789",
  "show_type": "xfactor",
  "remaining_free_votes": 4,
  "contestant_total": 23501,
  "rank": 2,
  "message": "Vote counted! Dara now has 23,501 votes",
  "show_specific_result": {
    "category_rank": 1, // XFactor
    "team_position": 2   // Voice
  }
}
```

### 4.4 Results & Analytics

**Live Results Display:**
```
┌─────────────────────────┐
│ Live Voting Results     │
│ Updated every 30 sec    │
├─────────────────────────┤
│ 1. 👑 Sophia            │
│    ████████████ 45.2%   │
│    54,234 votes         │
│                         │
│ 2. 🥈 Dara              │
│    ████████ 38.1%       │
│    45,678 votes         │
│                         │
│ 3. 🥉 Virak             │
│    ████ 16.7%           │
│    20,012 votes         │
│                         │
│ Total Votes: 119,924    │
│ Your Impact: 0.004%     │
└─────────────────────────┘
```

---

## 5. Payment Processing

### 5.1 ABA PayWay Integration

**QR Code Payment Flow:**
1. User selects ABA PayWay
2. App generates unique QR code
3. User scans with ABA Mobile app
4. Confirms payment in ABA app
5. Hang Meas app receives confirmation
6. Tickets issued instantly

**Example QR Display:**
```
┌─────────────────────────┐
│ Scan to Pay with ABA    │
│                         │
│      [QR CODE]          │
│                         │
│ Amount: $210.00         │
│ Order: HM2024122001     │
│                         │
│ Expires in: 4:45        │
│                         │
│ Having trouble?         │
│ [Pay Another Way]       │
└─────────────────────────┘
```

### 5.2 Wing Payment

**Phone Number Payment:**
```
┌─────────────────────────┐
│ Pay with Wing           │
├─────────────────────────┤
│ Enter Wing Account      │
│ 📱 +855 __ ___ ___      │
│                         │
│ Amount: $210.00         │
│                         │
│ [Send Payment Request]  │
│                         │
│ You'll receive an SMS   │
│ to confirm payment      │
└─────────────────────────┘
```

**Confirmation Process:**
1. User enters Wing phone number
2. Wing sends PIN request SMS
3. User enters PIN in SMS reply
4. Payment processed
5. Both apps show confirmation

### 5.3 Payment Security

**Security Features:**
- 3D Secure for card payments
- Tokenization of card details
- SSL/TLS encryption
- PCI DSS compliance
- Fraud detection algorithms

**Failed Payment Handling:**
```
┌─────────────────────────┐
│ ⚠️ Payment Failed       │
├─────────────────────────┤
│ Insufficient funds      │
│                         │
│ Your seats are held for │
│ 10 minutes. Try again?  │
│                         │
│ [Try Different Method]  │
│ [Cancel Order]          │
└─────────────────────────┘
```

---

## 6. Digital Ticket Management

### 6.1 QR Code Tickets

**Ticket Display:**
```
┌─────────────────────────┐
│ Preap Sovath Live       │
│ Dec 25, 2024 7:00 PM    │
├─────────────────────────┤
│      [QR CODE]          │
│                         │
│ Ticket ID: HM123456     │
│ Seat: VIP A4            │
│ Gate: North Entrance    │
├─────────────────────────┤
│ Ticket Holder:          │
│ សុខ សុភា               │
│ Sok Sophia              │
├─────────────────────────┤
│ [Add to Apple Wallet]   │
│ [Share Ticket]          │
└─────────────────────────┘
```

**QR Code Features:**
- Dynamic QR (changes every 30 seconds)
- Works offline
- Contains encrypted ticket data
- Prevents screenshots fraud

### 6.2 Ticket Transfer

**Transfer Process:**
```
┌─────────────────────────┐
│ Transfer Ticket         │
├─────────────────────────┤
│ Send ticket to:         │
│ 📱 Phone: +855_______  │
│        OR               │
│ ✉️ Email: _________    │
│                         │
│ ⚠️ Once transferred,    │
│ you cannot reclaim      │
│ this ticket             │
│                         │
│ [Send Transfer]         │
└─────────────────────────┘
```

### 6.3 Entry Validation

**At Venue:**
1. Staff scans QR code
2. System validates:
   - Ticket authenticity
   - Not already used
   - Correct event/date
   - Valid seat
3. Green ✓ or Red ✗ display
4. Entry logged with timestamp

---

## 7. Push Notifications

### 7.1 Notification Types

**Event Reminders:**
```
🎫 Hang Meas Tickets
Tomorrow: Preap Sovath Live at 7PM
Don't forget! Gate opens at 6PM
[View Ticket]
```

**Talent Show Voting:**
```
🗳️ XFactor Voting LIVE NOW!
Semi-Final Round 2 has started
You have 5 free votes
[Vote Now]
```

```
🌟 CGT Golden Buzzer Alert!
Someone just got the Golden Buzzer!
Voting opens in 30 minutes
[Watch Now]
```

```
🎤 Voice Blind Auditions!
New coaches, new voices
Chair turns happening now
[Join Live]
```

**New Events:**
```
🎤 New Event Alert!
Sokun Nisa announces concert tour
Exclusive presale starts in 2 hours
[Get Tickets]
```

**Price Drops:**
```
💰 Price Drop Alert!
Comedy Show tickets now 30% off
Limited time: 24 hours only
[Shop Now]
```

### 7.2 Personalization

**Smart Notifications Based On:**
- Favorite artists
- Past purchases
- Browsing history
- Location
- Language preference

**Frequency Control:**
- Maximum 3 per day
- Quiet hours (10 PM - 8 AM)
- User can customize

---

## 8. Admin Dashboard

### 8.1 Event Management

**Event Creation Form:**
```
Event Details
├── Basic Information
│   ├── Event Title (KM/EN)
│   ├── Category
│   ├── Date & Time
│   └── Venue Selection
├── Ticketing
│   ├── Ticket Types
│   ├── Pricing Tiers
│   ├── Capacity
│   └── Sale Period
├── Content
│   ├── Description (KM/EN)
│   ├── Images/Videos
│   ├── Artist Info
│   └── Terms & Conditions
└── Marketing
    ├── Featured Status
    ├── Promotional Codes
    ├── Early Bird Settings
    └── Partner Allocations
```

### 8.2 Sales Analytics

**Real-time Dashboard:**
```
┌─────────────────────────┐
│ Today's Performance     │
├─────────────────────────┤
│ Revenue: $45,230 ▲23%   │
│ Tickets Sold: 1,456     │
│ Active Users: 3,421     │
│                         │
│ Top Events:             │
│ 1. XFactor Final (623)  │
│ 2. CGT Semi-Final (456) │
│ 3. Voice Battles (389)  │
│ 4. NYE Concert (234)    │
│                         │
│ [Detailed Analytics →]  │
└─────────────────────────┘
```

### 8.3 Customer Support Tools

**Ticket Management:**
- Void/refund tickets
- Reissue lost tickets
- Transfer between customers
- Upgrade/downgrade seats
- View purchase history
- Handle disputes

**Communication:**
- Send targeted notifications
- Email campaigns
- SMS broadcasts
- In-app messages
- Support ticket system

---

## Venue Capacity Reference

### Typical Cambodia Venue Sizes
To ensure realistic planning and capacity management:

**Large Venues (3,000-5,000 capacity)**
- Olympic Stadium (Concert Area): 5,000
- Koh Pich Convention Center: 4,000
- Diamond Island Hall: 3,500

**Medium Venues (1,500-3,000 capacity)**
- Koh Pich Theatre: 2,500
- RUPP Auditorium: 2,000
- Chaktomuk Theatre: 1,800
- Hard Rock Cafe: 1,500

**Small Venues (500-1,500 capacity)**
- Factory Phnom Penh: 1,000
- Cloud Lounge: 800
- Exchange Bar: 600
- Show Box: 500

**Event Categories by Venue Size**
- Superstar Concerts: Large venues (3,000-5,000)
- Popular Artists: Medium venues (1,500-3,000)
- Rising Stars: Small venues (500-1,500)
- Exclusive/VIP Events: Small venues with premium pricing

---

## Multi-Show Streaming Infrastructure

> **Comprehensive Streaming Guide**: For detailed streaming infrastructure, encoding solutions, CDN setup, and technical implementation for all talent shows and live events, see [STREAMING_INFRASTRUCTURE.md](../technical/STREAMING_INFRASTRUCTURE.md)

> **Show-Specific Architecture**: For talent show implementations and voting mechanics, see [TALENT_SHOWS_FRAMEWORK.md](../TALENT_SHOWS_FRAMEWORK.md)

## Technical Implementation Notes

### API Response Examples

**Event List Response:**
```json
{
  "events": [
    {
      "id": "evt_123",
      "title": {
        "km": "ការប្រគុំតន្ត្រីឆ្នាំថ្មី",
        "en": "New Year Concert"
      },
      "date": "2024-12-31T19:00:00+07:00",
      "venue": {
        "id": "ven_456",
        "name": "Koh Pich Theatre",
        "capacity": 2500
      },
      "price_range": {
        "min": 15,
        "max": 150,
        "currency": "USD"
      },
      "availability": {
        "status": "on_sale",
        "percentage_sold": 67
      },
      "thumbnail": "https://cdn.hangmeas.com/events/123/thumb.jpg"
    }
  ],
  "pagination": {
    "page": 1,
    "per_page": 20,
    "total": 145
  }
}
```

**Ticket Purchase Response:**
```json
{
  "order": {
    "id": "ord_789",
    "status": "completed",
    "tickets": [
      {
        "id": "tkt_001",
        "qr_code": "https://api.hangmeas.com/qr/tkt_001",
        "seat": "A4",
        "type": "VIP",
        "holder_name": "Sok Sophia"
      }
    ],
    "total_amount": 210.00,
    "payment_method": "aba_payway",
    "created_at": "2024-12-20T15:30:00Z"
  },
  "confirmation_sent_to": [
    "phone:+855123456789",
    "email:sophia@example.com"
  ]
}
```

### Error Handling

**Common Error Responses:**
```json
{
  "error": {
    "code": "SOLD_OUT",
    "message": {
      "km": "សំបុត្រអស់ហើយ",
      "en": "Tickets are sold out"
    },
    "details": {
      "event_id": "evt_123",
      "requested_quantity": 2
    }
  }
}
```

### Performance Considerations

1. **Image Optimization:**
   - Thumbnail: 300x200px (50KB max)
   - Full image: 1200x800px (500KB max)
   - WebP format with JPEG fallback

2. **Caching Strategy:**
   - Event list: 5 minutes
   - Event details: 1 minute
   - User profile: 30 minutes
   - Static content: 24 hours

3. **Load Testing Targets:**
   - Support 50,000 concurrent users
   - Handle 1,000 orders/minute
   - API response time < 200ms
   - 99.9% uptime SLA