# Flow Diagrams and System Architecture

> **Multi-Show Support**: This architecture supports multiple talent shows (XFactor, Cambodia's Got Talent, The Voice Cambodia) through a unified framework. See [TALENT_SHOWS_FRAMEWORK.md](../TALENT_SHOWS_FRAMEWORK.md) for show-specific implementations.

## 1. Complete User Registration Flow

### Flow Diagram: New User Registration
```
┌─────────────────┐
│   User Opens    │
│      App        │
└────────┬────────┘
         │
         ▼
┌─────────────────┐     ┌─────────────────┐
│  Splash Screen  │     │   App Checks    │
│   (2 seconds)   │────▶│  First Launch   │
└─────────────────┘     └────────┬────────┘
                                 │
                    ┌────────────┴────────────┐
                    │                         │
                    ▼                         ▼
          ┌─────────────────┐       ┌─────────────────┐
          │  First Time     │       │ Returning User  │
          │  Show Onboarding│       │ Go to Home      │
          └────────┬────────┘       └─────────────────┘
                   │
                   ▼
          ┌─────────────────┐
          │ Language Select │
          │ 🇰🇭 ខ្មែរ           │
          │ 🇬🇧 English      │
          └────────┬────────┘
                   │
                   ▼
          ┌─────────────────┐
          │ Create Account  │
          │ or Browse Guest │
          └────────┬────────┘
                   │
        ┌──────────┴──────────┐
        │                     │
        ▼                     ▼
┌─────────────────┐  ┌─────────────────┐
│ Phone Number    │  │ Social Login    │
│ Registration    │  │ FB/Google/Apple │
└────────┬────────┘  └────────┬────────┘
         │                    │
         ▼                    │
┌─────────────────┐           │
│ Send OTP SMS    │           │
│ to +855XXXXXXX  │           │
└────────┬────────┘           │
         │                    │
         ▼                    │
┌─────────────────┐           │
│ Verify OTP Code │           │
│ [_][_][_][_]    │           │
└────────┬────────┘           │
         │                    │
         └──────────┬─────────┘
                    │
                    ▼
          ┌──────────────-───┐
          │ Complete Profile │
          │ - Name (KH/EN)   │
          │ - Date of Birth  │
          │ - Location       │
          │ - Preferences    │
          └────────┬───────-─┘
                   │
                   ▼
          ┌─────────────────┐
          │ Enable Features │
          │ ☐ Notifications │
          │ ☐ Location      │
          │ ☐ Biometrics    │
          └────────┬────────┘
                   │
                   ▼
          ┌─────────────────┐
          │ Welcome Screen  │
          │ Tutorial Cards  │
          │ - How to buy    │
          │ - XFactor voting│
          │ - Rewards       │
          └────────┬────────┘
                   │
                   ▼
          ┌─────────────────┐
          │   Home Screen   │
          │  Logged In User │
          └─────────────────┘
```

### Detailed Registration Steps

#### Step 1: Phone Number Validation
```javascript
// Phone number input validation
{
  "validation_rules": {
    "format": "^\\+855[0-9]{8,9}$",
    "carriers": ["Smart", "Cellcard", "Metfone"],
    "blacklist_check": true,
    "rate_limit": "3_attempts_per_hour"
  }
}

// Example flow
User enters: 012345678
System converts: +855 12 345 678
Validates: ✓ Smart carrier detected
Sends OTP: "Your code is 4821"
```

#### Step 2: OTP Verification Process
```
┌─────────────────────────────┐
│     Verify Your Number      │
├─────────────────────────────┤
│  We sent a code to          │
│  +855 12 345 678            │
│                             │
│  Enter 4-digit code:        │
│  ┌───┐ ┌───┐ ┌───┐ ┌───┐    │
│  │ 4 │ │ 8 │ │ 2 │ │ 1 │    │
│  └───┘ └───┘ └───┘ └───┘    │
│                             │
│  Resend code in 0:45        │
│                             │
│  [Verify] [Change Number]   │
└─────────────────────────────┘

Backend process:
1. Generate random 4-digit code
2. Store in Redis with 5-min TTL
3. Send via SMS gateway
4. Track attempts (max 5)
5. Success: Create JWT token
```

## 2. Ticket Purchase Flow - Complete Journey

### High-Level Purchase Flow
```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│   Browse    │────▶│   Select    │────▶│    Seat     │
│   Events    │     │   Tickets   │     │  Selection  │
└─────────────┘     └─────────────┘     └──────┬──────┘
                                               │
┌─────────────┐     ┌─────────────┐     ┌──────▼──────┐
│  Success    │◀────│   Payment   │◀────│   Review    │
│  & Delivery │     │  Processing │     │    Order    │
└─────────────┘     └─────────────┘     └─────────────┘
```

### Detailed Seat Selection Process
```
                    Venue: Koh Pich Theatre
    ┌─────────────────────STAGE─────────────────────┐
    
    VIP Section ($100)
    A  [01][02][03][04][XX][XX][07][08][09][10]
    B  [01][02][XX][XX][XX][06][07][08][09][10]
    
    Premium Section ($50)
    C  [01][02][03][04][05][06][07][08][09][10]
    D  [01][02][03][04][05][06][07][08][09][10]
    E  [01][02][03][04][05][06][07][08][09][10]
    
    Standard Section ($25)
    F  [01][02][03][04][05][06][07][08][09][10]
    G  [01][02][03][04][05][06][07][08][09][10]
    H  [01][02][03][04][05][06][07][08][09][10]
    
    Legend: [XX] Sold  [##] Selected  [01] Available

User Interaction:
1. Pinch to zoom venue map
2. Tap section to expand
3. Tap seat to select (turns blue)
4. See photo from seat view
5. Confirm selection
```

### Payment Integration Flow
```
┌─────────────────────────────┐
│     Payment Selection       │
└──────────────┬──────────────┘
               │
     ┌─────────┴─────────┬─────────┬─────────┐
     │                   │         │         │
     ▼                   ▼         ▼         ▼
┌──────────┐      ┌──────────┐ ┌──────────┐ ┌──────────┐
│   ABA    │      │  ACLEDA  │ │   Wing   │ │   Card   │
│  PayWay  │      │   XPay   │ │  Money   │ │  Payment │
└────┬─────┘      └────┬─────┘ └────┬─────┘ └────┬─────┘
     │                 │            │            │
     ▼                 ▼            ▼            ▼
┌──────────┐      ┌──────────┐ ┌──────────┐ ┌──────────┐
│Generate  │      │  Direct  │ │   SMS    │ │ 3D Secure│
│ QR Code  │      │  Debit   │ │  Verify  │ │  Check   │
└────┬─────┘      └────┬─────┘ └────┬─────┘ └────┬─────┘
     │                 │            │            │
     └─────────┬───────┴────────────┴────────────┘
               │
               ▼
        ┌──────────────┐
        │   Payment    │
        │   Gateway    │
        │  Processing  │
        └──────┬───────┘
               │
         ┌─────┴─────┐
         │           │
         ▼           ▼
    ┌─────────┐ ┌─────────┐
    │ Success │ │ Failed  │
    │  Issue  │ │  Retry  │
    │ Tickets │ │ Options │
    └─────────┘ └─────────┘
```

## 3. Multi-Show Live Voting System Architecture

> **Universal Voting Framework**: Supports XFactor categories, CGT golden buzzer mechanics, and Voice coach teams through configurable voting rules.

### Multi-Show Real-Time Voting Infrastructure
```
┌─────────────────────────────────────────────────────┐
│                   TV Broadcast                      │
│                  Signal (Live)                      │
└───────────────────────┬─────────────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────────┐
│              Broadcast Sync Server                  │
│         (Timestamps & Event Triggers)               │
└───────────────────────┬─────────────────────────────┘
                        │
        ┌───────────────┼───────────────┐
        │               │               │
        ▼               ▼               ▼
┌───────────────┐ ┌───────────────┐ ┌───────────────┐
│  Mobile App   │ │   Web App     │ │  SMS Gateway  │
│   (iOS/And)   │ │  (Browser)    │ │   (Voting)    │
└───────┬───────┘ └───────┬───────┘ └───────┬───────┘
        │                 │                 │
        └─────────────────┴─────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────┐
│                 API Gateway                         │
│            (Load Balancer + Auth)                   │
└───────────────────────┬─────────────────────────────┘
                        │
        ┌───────────────┼───────────────┐
        │               │               │
        ▼               ▼               ▼
┌───────────────┐ ┌──────────-─────┐ ┌───────────────┐
│ Voting Service│ │ Results Service│ │ Anti-Fraud    │
│  (Write Heavy)│ │  (Read Heavy)  │ │   Service     │
└───────┬───────┘ └───────┬─-──────┘ └───────┬───────┘
        │                 │                  │
        ▼                 ▼                  ▼
┌───────────────┐ ┌───────────────┐ ┌───────────────┐
│     Redis     │ │  PostgreSQL   │ │   ML Model    │
│ (Vote Queue)  │ │  (Persistent) │ │ (Fraud Score) │
└───────────────┘ └───────────────┘ └───────────────┘
```

### Voting Window State Machine
```
┌────────────┐
│   IDLE     │ (No active show)
└─────┬──────┘
      │ Show starts
      ▼
┌────────────┐
│PERFORMANCES│ (Contestants performing)
└─────┬──────┘
      │ All performed
      ▼
┌────────────┐
│   READY    │ (Preparing to open voting)
└─────┬──────┘
      │ Host announces
      ▼
┌────────────┐
│   OPEN     │ (Accept votes - 30 min)
└─────┬──────┘
      │ Timer expires
      ▼
┌────────────┐
│  CLOSING   │ (Final 10 seconds warning)
└─────┬──────┘
      │
      ▼
┌────────────┐
│  COUNTING  │ (Processing final votes)
└─────┬──────┘
      │
      ▼
┌────────────┐
│  RESULTS   │ (Display winner/eliminated)
└─────┬──────┘
      │
      ▼
┌────────────┐
│   IDLE     │
└────────────┘
```

## 4. Event Discovery Algorithm

### Personalized Event Recommendation Engine
```
┌─────────────────────────────────────┐
│         User Profile Data           │
│  - Past purchases                   │
│  - Browsing history                 │
│  - Favorite artists                 │
│  - Location                         │
│  - Age/Demographics                 │
└──────────────┬──────────────────────┘
               │
               ▼
┌─────────────────────────────────────┐
│      Recommendation Engine          │
├─────────────────────────────────────┤
│ 1. Collaborative Filtering          │
│    - Users who bought X also Y      │
│                                     │
│ 2. Content-Based Filtering          │
│    - Similar artists/genres         │
│                                     │
│ 3. Trending Algorithm               │
│    - Velocity of sales              │
│    - Social media buzz              │
│                                     │
│ 4. Location-Based                   │
│    - Nearby venues                  │
│    - Travel time consideration      │
└──────────────┬──────────────────────┘
               │
               ▼
┌─────────────────────────────────────┐
│        Scoring & Ranking            │
│  Score = 0.3*CF + 0.3*CBF +         │
│          0.2*Trend + 0.2*Location   │
└──────────────┬──────────────────────┘
               │
               ▼
┌─────────────────────────────────────┐
│      Personalized Feed              │
│  1. Preap Sovath Concert (95%)      │
│  2. Comedy Night (87%)              │
│  3. XFactor Final (84%)             │
│  4. Jazz Festival (76%)             │
└─────────────────────────────────────┘
```

## 5. Digital Ticket Security System

### QR Code Generation and Validation
```
Ticket Purchase                    Event Entry
       │                              │
       ▼                              ▼
┌─────────────┐                ┌─────────────┐
│  Generate   │                │  Scan QR    │
│  Unique ID  │                │   Code      │
└──────┬──────┘                └──────┬──────┘
       │                              │
       ▼                              ▼
┌─────────────┐                ┌─────────────┐
│  Encrypt    │                │  Decrypt    │
│   Data:     │                │  & Parse    │
│ -Ticket ID  │                │   Data      │
│ -Event ID   │                └──────┬──────┘
│ -User ID    │                       │
│ -Timestamp  │                       ▼
└──────┬──────┘                ┌─────────────┐
       │                       │  Validate:  │
       ▼                       │ -Not used   │
┌─────────────┐                │ -Correct    │
│  Generate   │                │  event      │
│  QR with    │                │ -Valid time │
│  30s refresh│                └──────┬──────┘
└──────┬──────┘                       │
       │                              ▼
       ▼                       ┌─────────────┐
┌─────────────┐                │   Grant/    │
│   Display   │                │   Deny      │
│  to User    │                │   Entry     │
└─────────────┘                └─────────────┘

Security Layers:
1. AES-256 encryption
2. HMAC signature
3. Time-based rotation
4. Device binding
5. Offline validation capability
```

## 6. Payment Processing Detailed Flow

### ABA PayWay Integration
```
User Selects ABA PayWay
         │
         ▼
┌───────────────────┐
│ App Request to    │
│ Payment Gateway   │
│ {                 │
│   amount: 52.50,  │
│   currency: USD,  │
│   order_id: xxx   │
│ }                 │
└─────────┬─────────┘
          │
          ▼
┌───────────────────┐        ┌──────────────────┐
│ Gateway Response  │        │ User's ABA App   │
│ {                 │        │                  │
│   qr_code: "...", │───────▶│ 1. Open ABA app  │
│   reference: xxx, │        │ 2. Scan QR       │
│   expires: 300s   │        │ 3. Confirm amount│
│ }                 │        │ 4. Enter PIN     │
└─────────┬─────────┘        └────────┬─────────┘
          │                           │
          ▼                           ▼
┌───────────────────┐        ┌──────────────────┐
│ Polling Status    │◀───────│ ABA Process      │
│ Every 2 seconds   │        │ Payment          │
└─────────┬─────────┘        └──────────────────┘
          │
          ▼
┌───────────────────┐
│ Payment Complete  │
│ Issue tickets     │
└───────────────────┘
```

### Wing Money Flow
```
┌───────────────────────-─────┐
│   User Enters Wing Number   │
│      012 345 678            │
└─────────────┬──────────-────┘
              │
              ▼
┌──────────────────────────-──┐
│  System Sends Wing Request  │
│  - Merchant: Hang Meas      │
│  - Amount: $52.50           │
│  - To: 012345678            │
└─────────────┬─────────────-─┘
              │
              ▼
┌─────────────────────-───────┐
│    User Receives SMS        │
│ "Pay $52.50 to Hang Meas?   │
│  Reply with PIN"            │
└─────────────┬────────────-──┘
              │
              ▼
┌───────────────────────-─────┐
│    User Replies: 1234       │
└─────────────┬──────────-────┘
              │
              ▼
┌────────────────────-────────┐
│  Wing Confirms Payment      │
│  Webhook to Hang Meas       │
└─────────────┬───────-───────┘
              │
              ▼
┌──────────────────────-──────┐
│    Tickets Issued           │
│    SMS Confirmation         │
└───────────────────────-─────┘
```

## 7. Push Notification Strategy

### Notification Decision Tree
```
┌─────────────────────┐
│  Trigger Event      │
│  Occurs             │
└──────────┬──────────┘
           │
           ▼
┌─────────────────────┐
│  Check User Prefs   │
│  - Enabled?         │
│  - Category allowed?│
│  - Time restrictions│
└──────────┬──────────┘
           │
     ┌─────┴─────┐
     │ Allowed?  │
     └─────┬─────┘
       No  │  Yes
       │   │
       ▼   ▼
   ┌──────┐ ┌─────────────────┐
   │ Skip │ │ Check Priority  │
   └──────┘ └────────┬────────┘
                     │
              ┌──────┴──────┐
              │ Priority?   │
              └──────┬──────┘
                     │
         ┌───────────┼───────────┐
         │           │           │
         ▼           ▼           ▼
    ┌─────────┐ ┌─────────┐ ┌─────────┐
    │Critical │ │  High   │ │ Normal  │
    │Send Now │ │ Bundle  │ │  Queue  │
    └─────────┘ └─────────┘ └─────────┘

Priority Levels:
- Critical: Payment issues, event changes
- High: Voting windows, last tickets
- Normal: New events, recommendations
```

## 8. Admin Dashboard Analytics

### Real-Time Sales Dashboard
```
┌────────────────────────────────────────-─────────┐
│             Live Sales Monitor                   │
├─────────────────────────────────────────-────────┤
│                                                  │
│  Current Minute: $1,234  ▲23% vs last hour       │
│  ████████████████████████ 78 transactions        │
│                                                  │
│  Last Hour Performance                           │
│  20:00 ████████████ $12,340                      │
│  21:00 ██████████████████ $18,450                │
│  22:00 ████████ $8,230                           │
│                                                  │
│  Top Selling Events (Live)                       │
│  1. XFactor Final    ████████ 234 tickets/min    │
│  2. NYE Concert      █████ 123 tickets/min       │
│  3. Comedy Show      ██ 45 tickets/min           │
│                                                  │
│  System Health                                   │
│  API Response: 89ms  ✓                           │
│  Payment Success: 97.3%                          │
│  Active Users: 3,421                             │
└────────────────────────────────────────────────-─┘
```

## 9. Error Recovery Mechanisms

### Payment Failure Recovery
```
Payment Attempt
      │
      ▼
┌─────────────┐
│  Process    │
│  Payment    │
└──────┬──────┘
       │
   ┌───┴───┐
   │Failed?│
   └───┬───┘
       │ Yes
       ▼
┌─────────────┐
│Check Reason │
└──────┬──────┘
       │
       ├─── Insufficient Funds ──▶ Suggest smaller package
       │
       ├─── Network Timeout ────▶ Retry with saved data
       │
       ├─── Bank Declined ──────▶ Try different method
       │
       └─── Technical Error ────▶ Hold seats + support

Seat Holding Logic:
- Initial hold: 10 minutes
- Failed payment: +5 minutes
- Max extensions: 2
- Release after 20 minutes total
```

## 10. Offline Mode Architecture

### Offline Data Sync Strategy
```
┌────────────────────────┐
│    Online Mode         │
│  Full functionality    │
└───────────┬────────────┘
            │ Connection Lost
            ▼
┌────────────────────────┐
│   Offline Mode         │
│  - View purchased      │
│    tickets             │
│  - Browse cached       │
│    events              │
│  - Queue actions       │
└───────────┬────────────┘
            │
            ▼
┌────────────────────────┐
│  Local Storage         │
│  - SQLite DB           │
│  - Encrypted tickets   │
│  - Event cache         │
│  - Action queue        │
└───────────┬────────────┘
            │ Connection Restored
            ▼
┌────────────────────────┐
│   Sync Process         │
│  1. Upload queued      │
│  2. Download updates   │
│  3. Resolve conflicts  │
│  4. Update UI          │
└────────────────────────┘
```